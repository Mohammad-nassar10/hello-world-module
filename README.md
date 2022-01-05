# Hello World Module
## A Helm Chart for an example Fybrik module

## Introduction

This helm chart defines a common structure to deploy a Kubernetes job for an Fybrik module.

The configuration for the chart is in the values file.

## Prerequisites

- Kubernetes cluster 1.10+
- Helm 3.0.0+

## Installation

### Modify values in Makefile

In `Makefile`:
- Change `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `DOCKER_HOSTNAME`, `DOCKER_NAMESPACE`, `DOCKER_TAGNAME`, `DOCKER_IMG_NAME`, and `DOCKER_CHART_IMG_NAME` to your own preferences.

### Build Docker image for Python application
```bash
make docker-build
```

### Push Docker image to your preferred container registry
```bash
make docker-push
```

### Configure the chart
- When testing the chart, configure settings by editing the `values.yaml` directly.
- Modify repository in `values.yaml` to your preferred Docker image. 
- Modify copy/read action as needed with appropriate values.
- At runtime, the `fybrik-manager` will pass in the copy/read values to the module so you can leave them blank in your final chart. 

## Register as a Fybrik module

To register HWM as a Fybrik module apply `hello-world-module.yaml` to the fybrik-system namespace of your cluster.

To install the latest release run:

```bash
kubectl apply -f https://github.com/fybrik/hello-world-module/releases/latest/download/hello-world-module.yaml -n fybrik-system
```

### Version compatbility matrix

| Fybrik           | HWM     | Command
| ---              | ---     | ---
| 0.5.x            | 0.5.x   | `https://github.com/fybrik/hello-world-module/releases/download/v0.5.0/hello-world-module.yaml`
| master           | main    | `https://raw.githubusercontent.com/fybrik/hello-world-module/main/hello-world-module.yaml`



### Login to Helm registry
```bash
make helm-login
```

### Lint and install Helm chart
```bash
make helm-verify
```

### Push the Helm chart

```bash
make helm-chart-push
```

## Uninstallation
```bash
make helm-uninstall
```

## Deploy and test Fybrik module

Follow this section to deploy and test the module on a single cluster.

### Before you begin

Install Fybrik using the [Quick Start](https://fybrik.io/v0.5/get-started/quickstart/) guide. This sample assumes the use of the built-in catalog, Open Policy Agent (OPA) and flight module.

### Deploy DataShim

Deploy [datashim](https://github.com/datashim-io/datashim) on the cluster:

```bash
   kubectl apply --validate=false -f https://raw.githubusercontent.com/datashim-io/datashim/master/release-tools/manifests/dlf.yaml
   kubectl wait --for=condition=ready pod -n dlf --all --timeout=120s
```

### Deploy Fybrik module

Deploy `FybrikModule` in `fybrik-system` namespace:
```bash
kubectl create -f hello-world-module.yaml -n fybrik-system
```
### Test using Fybrik Notebook sample

1. Execute all the sections in [Fybrik Notebook sample](https://fybrik.io/v0.5/samples/notebook/) until `Deploy a Jupyter notebook` section.

1. Deploy the following `FybrikStorageAccount` and a secret resources. These resources are used by the Fybrik to allocate a new bucket for the copied resource.

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: bucket-creds
  namespace: fybrik-system
type: Opaque
stringData:
  access_key: "${ACCESS_KEY}"
  accessKeyID: "${ACCESS_KEY}"
  secret_key: "${SECRET_KEY}"
  secretAccessKey: "${SECRET_KEY}"
EOF
```
```bash
cat << EOF | kubectl apply -f -
apiVersion:   app.fybrik.io/v1alpha1
kind:         FybrikStorageAccount
metadata:
  name: storage-account
  namespace: fybrik-system
spec:
  endpoint:  "http://localstack.fybrik-notebook-sample.svc.cluster.local:4566"
  regions:
    - theshire
  secretRef:  bucket-creds
EOF
```
### Deploy Fybrik application which triggers module
Deploy `FybrikApplication` in `default` namespace:
```bash
kubectl apply -f fybrikapplication.yaml -n default
```
3.  Check if `FybrikApplication` successfully deployed:
```bash
kubectl get FybrikApplication -n default
kubectl describe FybrikApplication hello-world-module-test -n default
```

4.  Check if module was triggered in `fybrik-blueprints`:
```bash
kubectl get blueprint -n fybrik-blueprints
kubectl describe blueprint hello-world-module-test-default -n fybrik-blueprints
kubectl get job -n fybrik-blueprints
kubectl get pods -n fybrik-blueprints
```
If you are using the `hello-world-module` image, you should see this in the `kubectl logs` of your completed Pod:
```
$ kubectl logs rel1-hello-world-module-x2tgs

Hello World Module!

Connection name is s3

Connection format is parquet

Vault credential address is http://vault.fybrik-system:8200/

Vault credential role is module

Vault credential secret path is v1/fybrik/dataset-creds/%7B%22asset_id%22:%20%225067b64a-67bc-4067-9117-0aff0a9963ea%22%2C%20%22catalog_id%22:%20%220fd6ff25-7327-4b55-8ff2-56cc1c934824%22%7D

S3 bucket is fybrik-test-bucket

S3 endpoint is s3.eu-gb.cloud-object-storage.appdomain.cloud

COPY SUCCEEDED
```

