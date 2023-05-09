# Helm deploy WIP

VM: Ubuntu host, ubuntu guest

## Install dependancies

* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [docker](https://docs.docker.com/engine/install/ubuntu/)
* [helm](https://helm.sh/docs/intro/install/)
* [yq](https://github.com/mikefarah/yq/#install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Deploy MinIO


### Create Cluster

Create new `kind-config.yaml` with contents from [@cniackz](https://github.com/cniackz/public/wiki/How-to-install-MinIO-Using-Helm-in-Kubernetes#steps)
`kind create cluster --config ./kind-config.yaml`

### Install Operator and Tenant

Download:
```
curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.4.tgz
curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.4.tgz
```

* Install operator with Helm
* Install tenant with Helm

### Create and apply yaml Files

* Create service.yaml, operator.yaml, console-secret.yaml
* kubectl apply each

### Deploy a Tenant

* Helm command
* Expose tenant service for Console UI

### Access Operator and Console UIs

* kubectl port forward operator and tenant
* Expose Operator UI

Login:
* Get JWT
* Log into Operator with JWT
* Log into Console with minio/minio123

