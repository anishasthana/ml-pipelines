# Installing Kubeflow Pipelines with Tekton for Development

**Note:** You can get an all-in-one installation of Kubeflow on IBM Cloud or Minikube, including
[Kubeflow Pipelines with Tekton backend by following the instructions here](/guides/kfp_tekton_install.md).
If you would like to do it in development mode, or if you already have a Kubeflow deployment, please follow the instructions below.

## Table of Contents

<!-- START of ToC generated by running ./tools/mdtoc.sh sdk/README.md -->

  - [Prerequisites](#prerequisites)
  - [Install Tekton KFP with pre-built images](#install-tekton-kfp-with-pre-built-images)
  - [Development: Building from source code](#development-building-from-source-code)
    - [Prerequisites](#prerequisites)
    - [Frontend](#frontend)
    - [Backend](#backend)
    - [Minikube](#minikube)
  - [Python based visualizations](#python-based-visualizations)
  - [Unit test](#unit-test)
    - [API server](#api-server)
    - [Frontend](#frontend)
    - [DSL](#dsl)
  - [Integration test & E2E test](#integration-test--e2e-test)
  - [Troubleshooting](#troubleshooting)

<!-- END of ToC generated by running ./tools/mdtoc.sh sdk/README.md -->

## Prerequisites

1. [Install Tekton](https://github.com/tektoncd/pipeline/blob/master/docs/install.md#installing-tekton-pipelines-on-kubernetes).
    - Minimum version: `0.36.0`
2. Patch the Tekton configs for KFP
    ```shell
    kubectl patch cm feature-flags -n tekton-pipelines \
        -p '{"data":{"enable-custom-tasks": "true"}}'
    ```
3. Clone this repository
    ```
    git clone https://github.com/kubeflow/kfp-tekton.git
    cd kfp-tekton
    ```
4. `kubectl` client version `v1.20.0`+ to support the new kustomize plugins.

## Install Tekton KFP with pre-built images
1. Remove the old version of KFP from a previous Kubeflow deployment and webhooks if it exists on your cluster.
    ```shell
    kubectl delete -k manifests/kustomize/env/platform-agnostic
    kubectl delete MutatingWebhookConfiguration cache-webhook-kubeflow
    ```

2. Once the previous KFP deployment is removed, run the below command to deploy this modified Kubeflow Pipeline version with Tekton backend.
    ```shell
    kubectl create ns kubeflow # Skip this if kubeflow namespace aleady exists
    kubectl apply -k manifests/kustomize/env/platform-agnostic
    ```

    Check the new KFP deployment, it should take about 5 to 10 minutes. Please be aware that [cache-deployer-deployment](https://www.kubeflow.org/docs/pipelines/caching/) won't be running unless it's deployed on top of the entire [Kubeflow stack](https://www.kubeflow.org/docs/started/getting-started/) with [Cert Manager](https://cert-manager.io/docs/installation/kubernetes/).
    ```shell
    kubectl get pods -n kubeflow
    ```

    Once all the pods except Cache deployer are running, run the commands below to expose your KFP UI to a public endpoint.
    ```shell
    kubectl patch svc ml-pipeline-ui -n kubeflow -p '{"spec": {"type": "LoadBalancer"}}'
    kubectl get service ml-pipeline-ui -n kubeflow -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    ```

    Now that you have deployed Kubeflow Pipelines with Tekton, install the [KFP-Tekton SDK](/sdk/README.md) and follow
    the [KFP Tekton User Guide](/guides/kfp-user-guide) to start building your own pipelines.


## Development: Building from source code

### Prerequisites
1. [NodeJS 12 or above](https://nodejs.org/en/download/)
2. [Golang 1.13 or above](https://golang.org/dl/)
3. [Python 3.6 or above](https://www.python.org/downloads/)

### Frontend
The development instructions are under the [frontend](/frontend) directory. Below are the commands for building the frontend docker image.
```shell
cd frontend
npm run docker
```

### Backend
The KFP backend with Tekton uses a modified version of Kubeflow Pipelines api-server, persistent agent, and metadata writer.
1. To build these two images, clone this repository under the [GOPATH](https://golang.org/doc/gopath_code.html#GOPATH) and rename it to `pipelines`.
    ```shell
    cd $GOPATH/src/go/github.com/kubeflow
    git clone https://github.com/kubeflow/kfp-tekton.git
    mv kfp-tekton pipelines
    cd pipelines
    ```

2. For local binary builds, use the `go build` commands
   ```shell
   go build -o apiserver ./backend/src/apiserver
   go build -o agent ./backend/src/agent/persistence
   go build -o workflow ./backend/src/crd/controller/scheduledworkflow/*.go
   go build -o cache ./backend/src/cache/*.go
   ```

   Note: The metadata writer is written in Python, so the code will be compiled during runtime execution.

3. For Docker builds, use the below `docker build` commands
   ```shell
   DOCKER_REGISTRY="<fill in your docker registry here>"
   docker build -t ${DOCKER_REGISTRY}/api-server -f backend/Dockerfile .
   docker build -t ${DOCKER_REGISTRY}/persistenceagent -f backend/Dockerfile.persistenceagent .
   docker build -t ${DOCKER_REGISTRY}/metadata-writer -f backend/metadata_writer/Dockerfile .
   docker build -t ${DOCKER_REGISTRY}/scheduledworkflow -f backend/Dockerfile.scheduledworkflow .
   docker build -t ${DOCKER_REGISTRY}/cache-server -f backend/Dockerfile.cacheserver .
   ```

4. Push the images to registry and modify the Kustomization to use your own built images.

   Modify the `newName` under the `images` section in `manifests/kustomize/base/kustomization.yaml`.

   Now you can follow the [Install Tekton KFP with pre-built images](#install-tekton-kfp-with-pre-built-images) instructions to install your own KFP backend.



### Minikube
Minikube can pick your local Docker image so you don't need to upload to remote repository.

For example, to build API server image
```bash
$ docker build -t ml-pipeline-api-server -f backend/Dockerfile .
```

## Python based visualizations

Python based visualizations are a new method to visualize results within the
Kubeflow Pipelines UI. For more information about Python based visualizations
please visit the [documentation page](https://www.kubeflow.org/docs/pipelines/sdk/python-based-visualizations).
To create predefine visualizations please check the [developer guide](/backend/src/apiserver/visualization/README.md).

## Unit test

### API server
Run unit test for the API server
```bash
cd backend/src/ && go test ./...
```
### Frontend
TODO: add instruction

### DSL
```bash
pip install ./dsl/ --upgrade && python ./dsl/tests/main.py
pip install ./dsl-compiler/ --upgrade && python ./dsl-compiler/tests/main.py
```

## Integration test & E2E test

E2E test are done with on IBM Cloud Tekton Toolchain using [this](/.tekton) Tekton pipeline.

## Troubleshooting

**Q: How to access to the database directly?**

You can inspect mysql database directly by running:
```bash
kubectl run -it --rm --image=gcr.io/ml-pipeline/mysql:5.6 --restart=Never mysql-client -- mysql -h mysql
mysql> use mlpipeline;
mysql> select * from jobs;
```

**Q: How to inspect object store directly?**

Minio provides its own UI to inspect the object store directly:
```bash
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -l app=minio -o jsonpath='{.items[0].metadata.name}' -n ${NAMESPACE}) 9000:9000
Access Key:minio
Secret Key:minio123
```

**Q: I see an error of exceeding Github rate limit when deploying the system. What can I do?**

See [Ksonnet troubleshooting page](https://github.com/ksonnet/ksonnet/blob/master/docs/troubleshooting.md#github-rate-limiting-errors)

**Q: How do I check my API server log?**

API server logs are located at /tmp directory of the pod. To SSH into the pod, run:
```bash
kubectl exec -it -n ${NAMESPACE} $(kubectl get pods -l app=ml-pipeline -o jsonpath='{.items[0].metadata.name}' -n ${NAMESPACE}) -- /bin/sh
```
or
```bash
kubectl logs -n ${NAMESPACE} $(kubectl get pods -l app=ml-pipeline -o jsonpath='{.items[0].metadata.name}' -n ${NAMESPACE})
```

**Q: How to check my cluster status if I am using Minikube?**

Minikube provides dashboard for deployment
```bash
minikube dashboard
```
