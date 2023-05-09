# Helm deploy WIP

Much of this is scaffolding needed to execute the actual Helm commands.

Test environment:
* Ubuntu host
* Ubuntu guest VM

## Install dependancies

This is what was needed on a standard Ubuntu Server 22.04. Someone who already has a k8s deployment probably has these, perhaps except `yq`.

* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [docker](https://docs.docker.com/engine/install/ubuntu/)
* [helm](https://helm.sh/docs/intro/install/)
* [yq](https://github.com/mikefarah/yq/#install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Deploy MinIO

### Create Cluster

For testing:

* Create new `kind-config.yaml` with contents from [@cniackz](https://github.com/cniackz/public/wiki/How-to-install-MinIO-Using-Helm-in-Kubernetes#steps)
`kind create cluster --config ./kind-config.yaml`

### Install Operator

Before you begin the deployment, download the latest Operator and Tenant releases:
```
curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.4.tgz
curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.4.tgz
```

Deploy Operator with Helm
```
helm install \
   --namespace minio-operator \
   --create-namespace \
   minio-operator ./operator-5.0.4.tgz
```

#### Expose Operator UI

* Create `service.yaml`
  ```
  kubectl get service console -n minio-operator -o yaml > ~/service.yaml
  yq e -i '.spec.type="NodePort"' ~/service.yaml
  yq e -i '.spec.ports[0].nodePort = 30080' ~/service.yaml
  kubectl apply -f ~/service.yaml
  ```
* Create `operator.yaml`
  ```
  kubectl get deployment minio-operator -n minio-operator -o yaml > ~/operator.yaml
  yq -i -e '.spec.replicas |= 1' ~/operator.yaml
  kubectl apply -f ~/operator.yaml
  ```
* Create `console-secret.yaml`
  Copy lines 7-14 from `https://github.com/minio/operator/blob/master/resources/base/console-ui.yaml`
  `kubectl apply -f ./console-secret.yaml`

* Forward Operator port
  ```
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  ```
  
### Deploy a Tenant

* Helm command
  ```
  helm install \
  --namespace tenant-ns \
  --create-namespace \
  tenant-ns ./tenant-5.0.4.tgz
  ```
* Expose tenant service for Console UI
  ```
  kubectl --namespace tenant-ns port-forward svc/minio1-console 9443:9443
  ```
  
### Access Operator and Console UIs

* kubectl port forward operator and tenant
  ```
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  kubectl --namespace tenant-ns port-forward svc/minio-console 9443:9443
  ```
* Expose Operator UI

Login:
* Get JWT
* Log into Operator with JWT
* Log into Console with minio/minio123

