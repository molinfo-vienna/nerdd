# Nerdd

NERDD is a platform to create and run prediction models for computational chemistry. This repository
reflects the current state of the NERDD instances deployed at the COMP3D group (University of
Vienna) and provides the configuration files for setting up and running all components on a
Kubernetes cluster.

Part of the platform are:

* [nerdd-frontend](https://github.com/molinfo-vienna/nerdd-frontend): the user interface
* [nerdd-backend](https://github.com/molinfo-vienna/nerdd-frontend): the backend server
* [nerdd-module](https://github.com/molinfo-vienna/nerdd-module): basis for creating new prediction modules
* [nerdd-link](https://github.com/molinfo-vienna/nerdd-link): communication layer

A few example modules can be found here:

* [CYPstrate](https://github.com/molinfo-vienna/cypstrate)
* [CYPlebrity](https://github.com/molinfo-vienna/cyplebrity)
* [HitDexter](https://github.com/molinfo-vienna/hitdexter/)
* [NPScout](https://github.com/molinfo-vienna/np-scout)


## Installation

### Local preview with Docker Compose

```sh
docker compose -f https://github.com/molinfo-vienna/nerdd.git#main:stacks/minimum/compose.yaml up -d

# Open http://localhost:8080
```

Stop the stack:

```sh
docker compose -f https://github.com/molinfo-vienna/nerdd.git#main:stacks/minimum/compose.yaml down
```


### Local Preview with Tilt

Requires a running Kubernetes cluster (e.g. [MicroK8s](https://microk8s.io/docs/getting-started), 
[Minikube](https://minikube.sigs.k8s.io/docs/start/), [K3s](https://docs.k3s.io/quick-start), 
[k3d](https://k3d.io/stable/#installation), or 
[kind](https://kind.sigs.k8s.io/docs/user/quick-start/)) and [Tilt](https://tilt.dev/).

```sh
# Clone the repository
git clone https://github.com/molinfo-vienna/nerdd
cd nerdd

# Run tilt
tilt up

# Open Tilt UI at http://localhost:10350
# Open NERDD UI at https://localhost:8443 once all components have loaded
```

> [!TIP]
> By default, only the `cypstrate` prediction module is loaded. You can enable additional 
> prediction modules by editing the `apps` list in `Tiltfile`.


### Minimum cluster installation

To deploy a lightweight version of NERDD into a cluster, you need the following infrastructure 
components:

* [Strimzi](https://strimzi.io/)
  * option 1: official installation guide
  * option 2: `kubectl apply -k https://github.com/molinfo-vienna/nerdd//apps/strimzi/envs/infra?ref=main`
* [MinIO](https://operator.min.io/)
  * option 1: official installation guide
  * option 2: `kubectl apply -k https://github.com/molinfo-vienna/nerdd//apps/minio-operator/envs/minimum?ref=main`

Install the NERDD components (i.e. frontend, backend, job workers, database, kafka, s3 storage, cypstrate):

```sh
kubectl create namespace minimum
kubectl apply -k https://github.com/molinfo-vienna/nerdd//stacks/minimum?ref=main

# Forward service ports
kubectl -n minimum port-forward service/nerdd-proxy 8080:80

# Open http://localhost:8080
```

> [!TIP]
> By default, only the `cypstrate` prediction module is loaded. You can enable additional 
> prediction modules by running 
> `kubectl apply -k https://github.com/molinfo-vienna/nerdd//apps/<module>/envs/minimum?ref=main` 
> and replacing `<module>` (e.g. with `cyplebrity`, `np-scout`).


## Integrate a new prediction module

A new prediction module usually starts as ordinary code. For this demonstration, we use RDKit to 
calculate molecular weight:

```python
from rdkit.Chem import Descriptors

weight = Descriptors.MolWt(mol)
```

Wrap the calculation in a `nerdd-module` model:

```python
# molecular_weight.py
from nerdd_module import Model
from nerdd_module.preprocessing import Sanitize
from rdkit.Chem import Descriptors


class MolecularWeightModel(Model):
    def __init__(self):
        super().__init__([Sanitize()])

    def _predict_mols(self, mols):
        # yield a dictionary for each molecule containing predictions / computations
        for mol in mols:
            yield {"molecular_weight": Descriptors.MolWt(mol)}

    def _get_base_config(self):
        return {
            "name": "molecular-weight",
            "version": "0.1.0",
            "description": "Calculates molecular weight with RDKit.",
            # declare all fields that should be visible in prediction results
            "result_properties": [
                {
                    "name": "molecular_weight",
                    "visible_name": "Molecular weight",
                    "type": "float",
                }
            ],
        }
```

Create a `Dockerfile` next to `molecular_weight.py`. In it we need to include `nerdd-link`, which 
will run `MolecularWeightModel` as a service:

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY molecular_weight.py .

RUN apt-get update \
    && apt-get install -y --no-install-recommends libexpat1 libxext6 libxrender1 \
    && rm -rf /var/lib/apt/lists/* \
    && python -m venv /env \
    && /env/bin/pip install --no-cache-dir "nerdd-link==0.6.7"

ENV PYTHONPATH=/app
```

Push this image to a registry (e.g. [ghcr.io](https://ghcr.io)). For a quick, short-lived test, we 
use [OCIHub](https://ocihub.com):

```sh
# create a unique image name
# (the tag "2h" indicates how long this image will be available)
IMAGE="ocihub.com/molecular-weight-$(uuidgen):2h"
docker build -t "$IMAGE" .
docker push "$IMAGE"
echo "$IMAGE"
```

To add the image to a running NERDD cluster, create the following `kustomization.yaml` and replace 
`<IMAGE>` with the generated image URL obtained above. Here, we assume that the 
[`minimum` stack](#minimum-cluster-installation) is running. 

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: minimum

components:
  - https://github.com/molinfo-vienna/nerdd//apps/_common/nerdd-module/envs/minimum?ref=main

configMapGenerator:
  - name: app-config
    literals:
      - appName=molecular-weight
      - modelClass=molecular_weight.MolecularWeightModel
      - image=<IMAGE>  # REPLACE
      - cpuRequest=10m
      - memRequest=256Mi
      - cpuLimit=500m
      - memLimit=512Mi
```

Apply it from the directory containing `kustomization.yaml`:

```sh
kubectl apply -k .
```

After a short delay, the module appears on the local NERDD web page.


## Development


### Setup of our development cluster at univie

Prerequisites:

* Kubernetes cluster, e.g. [MicroK8s](https://microk8s.io/docs/getting-started), [Minikube](https://minikube.sigs.k8s.io/docs/start/), [K3s](https://docs.k3s.io/quick-start), [k3d](https://k3d.io/stable/#installation), or [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
* [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) deployment in the cluster
* at least 3 worker nodes (for Ceph Rook, Kafka, nerdd-backend, etc.)
* at least 3 **unpartitioned** disks on different worker nodes (for Ceph Rook)

```sh
# install nerdd application with all application sets
kubectl apply -f https://raw.githubusercontent.com/molinfo-vienna/nerdd/refs/heads/main/root.yaml

# make the rook-ceph namespace privileged:
kubectl label namespace rook-ceph \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

# bootstrap the shared GHCR credential (needs permission read:packages)
kubectl create namespace dev-credentials --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github_username> \
  --docker-password=<personal_access_token> \
  --docker-email=someone@example.com \
  -n dev-credentials

# git credentials for letting argocd-image-updater update the repository state to the repo
# (needs permissions repo & read:packages)
kubectl -n argocd create secret generic git-creds \
  --from-literal=githubAppID=<applicationid> \
  --from-literal=githubAppInstallationID=<installationid> \
  --from-literal=githubAppPrivateKey='-----BEGIN RSA PRIVATE KEY-----...-----END RSA PRIVATE KEY-----'
```

### Passwords

We use `external-secrets` to generate passwords for all dashboards and external tools in the
cluster. After deploying, the passwords can be retrieved using `kubectl`:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

### Switching the message broker

* tilt: set message_broker variable in `Tiltfile`
* argocd in dev environment: disable resources in the `kustomization.yaml` file of the currently
  used broker (e.g. `apps/kafka/envs/dev/kustomization.yaml`) and enable resources of the target
  message broker (e.g. `apps/rabbitmq/envs/dev/kustomization.yaml`) 

### Uninstall

```sh
kubectl delete applicationset dev-appset -n argocd
kubectl delete applicationset prod-appset -n argocd
```

```sh
argocd app delete nerdd-apps
# confirm that all data will be destroyed
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'
```

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
multiple git repositories for infrastructure and code. Specifically, it enables specifying the order 
of how apps are deployed.


## Troubleshooting

* Running `tilt up` leads to error messages of the form ```error: error upgrading connection: error 
dialing backend: tls: failed to verify certificate: x509: certificate is valid for 
<list of IP addresses>, not <IP address>```
  * fix: refresh Kubernetes certificates, e.g. using ```sudo microk8s refresh-certs --cert ca.crt```
* Tilt reports `Build Failed: kubernetes apply retry: timeout waiting for delete: kafkatopics.kafka.strimzi.io` meaning `tilt` is not able to clean up kafka topics. Try running:
  ```bash
  kubectl patch kafkatopic/jobs -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch kafkatopic/results -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch kafkatopic/system -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch kafkatopic/logs -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch kafkatopic/result-checkpoints -n local --type=merge  --patch '{"metadata":{"finalizers":[]}}'
  kubectl patch crd kafkatopics.kafka.strimzi.io -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```
* Tilt reports `Build Failed: kubernetes apply retry: timeout waiting for delete: scaledobjects.keda.sh`. Try running:
  ```bash
  kubectl patch crd scaledobjects.keda.sh -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```
* Tilt reports `error looking up service account external-secrets/external-secrets-cert-controller: serviceaccount "external-secrets-cert-controller" not found`:
  * VPN (e.g. ProtonVPN) might interfere with the cluster setup -> disable temporarily
* Tilt reports `filewatch configs:(Tiltfile): filewatch init: notify.Add("..."): no space left on device`
  * too many files are watched for changes
  * -> exclude a non-essential project from the main Tiltfile (e.g. cypstrate)
  * -> or: increase number of watchable files
* kubectl gives an error when trying to access a remote url: `error: failed to run '/snap/microk8s/9063/usr/bin/git fetch --depth=1 https://github.com/molinfo-vienna/nerdd main': /usr/lib/git-core/git-remote-https: symbol lookup error: /usr/lib/git-core/git-remote-https: undefined symbol: curl_global_trace, version CURL_GNUTLS_3
: exit status 128`
  * This happens when `kubectl` is installed via `snap`
  * -> option 1: install [standalone kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
  * -> option 2: clone the repository first and use local file paths

## Contribute

* install a Kubernetes cluster, e.g. [MicroK8s](https://microk8s.io/docs/getting-started), [Minikube](https://minikube.sigs.k8s.io/docs/start/), [K3s](https://docs.k3s.io/quick-start), [k3d](https://k3d.io/stable/#installation), or [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
* install [Tilt](https://tilt.dev/)
* clone repository
* `tilt up`
* visit localhost:10350 for tilt dashboard
* visit localhost:8443 for frontend application
* visit localhost:8443/api/ for backend api

## License

This repository is licensed under the [BSD 3-Clause License](LICENSE).
