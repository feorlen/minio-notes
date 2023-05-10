# Helm deploy WIP

Much of this is scaffolding needed to execute the actual Helm commands.

Test environment:
* Ubuntu 22.04 host
* Ubuntu 22.04 guest VM

## Install dependancies

This is what was needed on a standard Ubuntu Server 22.04. Someone who already has a k8s deployment probably has these, perhaps except `yq`.

* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [docker](https://docs.docker.com/engine/install/ubuntu/)
* [helm](https://helm.sh/docs/intro/install/)
* [yq](https://github.com/mikefarah/yq/#install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Deploy MinIO

### Create a Cluster

For testing, if one is not already available:

* Create new `kind-config.yaml` with contents from [@cniackz](https://github.com/cniackz/public/wiki/How-to-install-MinIO-Using-Helm-in-Kubernetes#steps)
* Create the cluster
  ```
  kind create cluster --config ./kind-config.yaml
  ```
* If needed, change permissions on the Docker socket
  ```
  sudo chmod 666 /var/run/docker.sock
  ```
  
### Install and Configure Operator

* Download the latest Operator and Tenant releases:
  ```
  curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.4.tgz
  curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.4.tgz
  ```

* Deploy Operator with Helm
  ```
  helm install \
     --namespace minio-operator \
     --create-namespace \
     minio-operator operator-5.0.4.tgz
  ```

* Create `service.yaml`
  ```
  kubectl get service console -n minio-operator -o yaml > service.yaml
  yq e -i '.spec.type="NodePort"' service.yaml
  yq e -i '.spec.ports[0].nodePort = 30080' service.yaml
  kubectl apply -f service.yaml
  ```

* Create `operator.yaml`
  ```
  kubectl get deployment minio-operator -n minio-operator -o yaml > operator.yaml
  yq -i -e '.spec.replicas |= 1' operator.yaml
  kubectl apply -f operator.yaml
  ```

* Create `console-secret.yaml`

  Lines 7-14 from `https://github.com/minio/operator/blob/master/resources/base/console-ui.yaml`
  ```
  apiVersion: v1
  kind: Secret
  metadata:
    name: console-sa-secret
    namespace: minio-operator
    annotations:
      kubernetes.io/service-account.name: console-sa
  type: kubernetes.io/service-account-token
  ```
  ```
  kubectl apply -f console-secret.yaml
  ```
  
### Deploy a Tenant

* Helm command
  ```
  helm install \
  --namespace tenant-ns \
  --create-namespace \
  tenant-ns tenant-5.0.4.tgz
  ```
* Expose tenant service for Console UI
  ```
  kubectl --namespace tenant-ns port-forward svc/myminio-console 9443:9443
  ```
  
### Enable the Operator and Console web UIs

* Expose the Operator and tenant ports
  ```
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  kubectl --namespace tenant-ns port-forward svc/minio-console 9443:9443
  ```

Log into Operator and Console:
* Get JWT
  ```
  SA_TOKEN=$(k -n minio-operator  get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode)
echo $SA_TOKEN
  ```
* Log into the Operator UI with the token
* Log into the MinIO Console with minio/minio123
