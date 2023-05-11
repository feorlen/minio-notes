# Helm deploy WIP

Much of this is scaffolding needed to execute the actual Helm commands.

Test environment:

* Ubuntu 22.04 host
* Ubuntu 22.04 guest VM

Port forwarding needed to access host/guest not covered. In my case host is a physical machine, but headless.

## Install dependancies

This is what was needed on a standard Ubuntu Server 22.04. Someone who already has a k8s deployment probably has these, perhaps except `yq`. For docs, reference its docs for `kubectl` commands.

* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [docker](https://docs.docker.com/engine/install/ubuntu/)
* [helm](https://helm.sh/docs/intro/install/)
* [yq](https://github.com/mikefarah/yq/#install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Deploy MinIO

### Create a Cluster

For testing, if one is not already available.

**What is the minimum number of nodes for a viable test? Are four workers required?**

* Create new `kind-config.yaml` with contents from [@cniackz](https://github.com/cniackz/public/wiki/How-to-install-MinIO-Using-Helm-in-Kubernetes#steps)
* Create the cluster
  ```
  kind create cluster --config ./kind-config.yaml
  ```
* If the above fails with `permission denied`, change permissions on the Docker socket
  ```
  sudo chmod 666 /var/run/docker.sock
  ```
  **When is this needed and why it isn't correct already?**
  
### Install and Configure Operator

* Download the latest Operator and Tenant releases:
  ```
  curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.4.tgz
  curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.4.tgz
  ```
  **I think just the `Chart.yaml` file is the Helm chart. What are the other files?**

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
  **What is this warning?**
  ```
  Warning: resource services/console is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply.
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
  
* Expose the Operator port
  ```
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  ```
  **Can this command be run in the background? What is the correct method for a production environment?**
  
  **Sometimes I get this error. What is it and how to resolve?**
  ```
  error: unable to forward port because pod is not running. Current status=Pending
  ```
  
* Log into Operator:
  * Get JWT
    ```
    SA_TOKEN=$(kubectl -n minio-operator  get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode)
    echo $SA_TOKEN
    ```
  * Go to http://localhost:9090
  * Log in with the token

### Deploy a Tenant

* Helm command
  ```
  helm install \
  --namespace tenant-ns \
  --create-namespace \
  tenant-ns tenant-5.0.4.tgz
  ```

* Expose the tenant port
  ```
  kubectl --namespace tenant-ns port-forward svc/myminio-console 9443:9443
  ```

* Log into the MinIO Console with minio/minio123

  **Is this a default login? How do you know to use this?**

## What to document (proposed)

* Install Operator with `helm` command and downloaded chart tgz
* Create `service.yaml`, `operator.yaml`, `console-secret.yaml`
* `port-forward svc/console 9090:9090` to permit logging into Operator
* Get the JWT and login to Operator
* Install a tenant with `helm` command and downloaded chart tgz
* `port-forward svc/myminio-console 9443:9443` to permit logging into the MinIO Console
* Login to the MinIO Console with minio/minio123

Note: other setup steps suitable for a personal wiki page showing entire process. How would this work with something other than kind/docker?
