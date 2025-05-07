# Nerdd

NERDD is a platform to create and run prediction models for computational chemistry. This repository 
provides the configuration files for setting up and running all components on a Kubernetes cluster.

## Prerequisites

* Kubernetes cluster
* ArgoCD deployed in that cluster
* optional (but recommended): 
  * at least 3 worker nodes (for Ceph Rook)
  * at least 3 **unpartitioned** disks on different worker nodes (for Ceph Rook)

## Installation

* Option 1: kubectl
```sh
kubectl apply -f https://raw.githubusercontent.com/molinfo-vienna/nerdd/refs/heads/main/root.yaml
```

* Option 1: ArgoCD CLI
```sh
argocd app create nerdd \
  --repo https://github.com/molinfo-vienna/nerdd \
  --path / \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

* Option 2: ArgoCD Web UI
  * (sign in)
  * create a new app (by clicking on "+ New App" on the top left)
  * set **application name** to **nerdd**
  * set **project name** to **default**
  * set **repository URL** to **https://github.com/molinfo-vienna/nerdd**
  * set **path** to **/**
  * set **cluster URL** to **https://kubernetes.default.svc**
  * set **namespace** to **default**


## Infrastructure

The NERDD infrastructure is managed by ArgoCD and the structure of this repository follows best 
practices presented in [this blog post](https://codefresh.io/blog/how-to-structure-your-argo-cd-repositories-using-application-sets/) and 
[the corresponding repository](https://github.com/kostis-codefresh/many-appsets-demo/tree/main). In 
a nutshell:
* the folder `apps` contains all components necessary to run NERDD on a kubernetes cluster,
* `appsets` specifies all different environments (e.g. infra, dev, prod), and
* `waves` orchestrates the order in which environments are installed (e.g. infra before dev),
* `root.yaml` is the entrypoint pointing to all waves available.

The concept of `waves` is an extension (not discussed in the blog post) in order to avoid having 
multiple git repositories for infrastructure and code. Especially, it enables specifying the order 
of how apps are deployed.

## Uninstall

```sh
argocd app delete nerdd
# confirm that all data will be destroyed
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'
```

## Troubleshooting

* Running tilt up leads to error messages of the form ```error: error upgrading connection: error 
dialing backend: tls: failed to verify certificate: x509: certificate is valid for 
<list of IP addresses>, not <IP address>```
  * fix: refresh Kubernetes certificates, e.g. using ```sudo microk8s refresh-certs --cert ca.crt```
* Tilt reports `Build Failed: kubernetes apply retry: timeout waiting for delete: kafkatopics.kafka.strimzi.io` meaning `tilt` is not able to clean up kafka topics. Try running:
  ```bash
  kubectl patch kafkatopic/jobs -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch kafkatopic/results -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch kafkatopic/system -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch crd kafkatopics.kafka.strimzi.io -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```
* Tilt reports `Build Failed: kubernetes apply retry: timeout waiting for delete: scaledobjects.keda.sh`. Try running:
  ```bash
  kubectl patch crd scaledobjects.keda.sh -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```

## Contribute

* Install docker
* Install a variant of kubernetes, e.g. microk8s, minikube, k3s, k3d or kind
* Install tilt
* `tilt up`
* visit localhost:10350 for tilt dashboard
* visit localhost:8443 for frontend application
* visit localhost:8443/api/ for backend api