## Building go applications using Google Cloud Build private go modules in Google Artifact Registry

Simple tutorial to build a golang container image using `Cloud Build` where the go module is read in from a private `Artifact Registry`

- [Artifact Registry for private Go Module](https://cloud.google.com/artifact-registry/docs/go)
- [Building go modules with Cloud Build](https://cloud.google.com/build/docs/building/build-go)

basically, you will 

- create a private go artifact repository (which i should emphasize is in private preview as of time of writing).

- we will upload custom go module to that repo

- use that module to build an app

- test that app locally and then create a local container image using that

- create a cloud build configuration which does the same.


The reason why i'm writing this is i wanted to figure something out new this morning..


>> note, this code is not supported by google

### Custom Calc

We will first define  custom go module called `example.com/salrashid123/calc` which does nothing but exports a function that adds two numbers

```golang
package calc

const ()

func Add(x int, y int) int {
	return x + y
}
```

We will then create a go app which references that remote (not local path reference):

```golang
package main

import (
	"flag"
	"example.com/salrashid123/calc"
	"github.com/golang/glog"
)

const ()

var ()

func main() {
	flag.Parse()
	
	glog.Infof("Add: %d\n", calc.Add(1,2))
}
```

### Setup

First we'll setup the artifact registry and what not.

```bash
export PROJECT_ID=`gcloud config get-value core/project`
export PROJECT_NUMBER=`gcloud projects describe $PROJECT_ID --format='value(projectNumber)'`

## create a custom service account for cloud build access to artifacts
gcloud iam service-accounts create gobuilder
gsutil mb gs://$PROJECT_ID\_cloudbuild
gsutil iam ch serviceAccount:gobuilder@$PROJECT_ID.iam.gserviceaccount.com:objectAdmin gs://$PROJECT_ID\_cloudbuild

## create go repo, again, this is on private preview
gcloud artifacts repositories create gar1 --repository-format=go --location=us-central1

gcloud artifacts repositories describe --location=us-central1  gar1

gcloud components install package-go-module

## upload the library to artifact registry
cd calc/
gcloud artifacts go upload \
    --repository=gar1 \
      --location=us-central1 \
      --module-path=example.com/salrashid123/calc \
      --version=v0.1.1 \
      --source=calc/

gcloud artifacts packages list --repository=gar1 --location=us-central1

gcloud artifacts packages describe example.com/salrashid123/calc --repository=gar1 --location=us-central1

## create docker repo
gcloud artifacts repositories create repo1 --repository-format=docker --location=us-central1

## add iam bindings to allow cloud build custom service account access to write loags, read from the go repo, write to docker repo
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:gobuilder@$PROJECT_ID.iam.gserviceaccount.com  \
  --role=roles/logging.logWriter

gcloud artifacts repositories add-iam-policy-binding gar1 \
    --location=us-central1  \
    --member=serviceAccount:gobuilder@$PROJECT_ID.iam.gserviceaccount.com  --role=roles/artifactregistry.reader

gcloud artifacts repositories add-iam-policy-binding repo1 \
    --location=us-central1  \
    --member=serviceAccount:gobuilder@$PROJECT_ID.iam.gserviceaccount.com  --role=roles/artifactregistry.writer
```

### Go build local 

Now build the go app locally

```bash
export GOPROXY=https://us-central1-go.pkg.dev/$PROJECT_ID/gar1,https://proxy.golang.org,direct
export GONOSUMDB=example.com/salrashid123/*
export GONOPROXY=github.com/GoogleCloudPlatform/artifact-registry-go-tools
GOPROXY=proxy.golang.org   go run github.com/GoogleCloudPlatform/artifact-registry-go-tools/cmd/auth@latest refresh

# to see the gory details
# export GODEBUG=http2debug=2

go mod download example.com/salrashid123/calc@v0.1.1

go run main.go   -v 20 -alsologtostderr
```

### Docker build Local

Build the docker image locally

```bash
export GOPROXY=https://us-central1-go.pkg.dev/$PROJECT_ID/gar1,https://proxy.golang.org,direct
export GONOSUMDB=example.com/salrashid123/*
export GONOPROXY=github.com/GoogleCloudPlatform/artifact-registry-go-tools
GOPROXY=proxy.golang.org   go run github.com/GoogleCloudPlatform/artifact-registry-go-tools/cmd/auth@latest refresh

cp $HOME/.netrc netrc

docker build -t  us-central1-docker.pkg.dev/$PROJECT_ID/repo1/add:server .

rm netrc
```

### Cloud Build remote

Build the container image using cloud build

```bash
gcloud beta builds submit --config=cloudbuild.yaml

docker run -t us-central1-docker.pkg.dev/$PROJECT_ID/repo1/add:server -v 20 -alsologtostderr
```

---
