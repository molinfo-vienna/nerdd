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



```sh
# Clone the repository
git clone https://github.com/molinfo-vienna/nerdd
cd nerdd

# Run tilt
tilt up

# Open Tilt UI at http://localhost:10350
# Open NERDD UI at https://localhost:8443 (after Tilt has loaded all components)
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
      - topic=molecular-weight-checkpoints
      - consumerGroup=predict-checkpoints-molecular-weight
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


## Installation

* Option 1: kubectl
```sh
kubectl apply -f https://raw.githubusercontent.com/molinfo-vienna/nerdd/refs/heads/main/root.yaml
```

* Option 2: ArgoCD CLI
```sh
argocd app create nerdd \
  --repo https://github.com/molinfo-vienna/nerdd \
  --path / \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

* Option 3: ArgoCD Web UI
  * (sign in)
  * create a new app (by clicking on "+ New App" on the top left)
  * set **application name** to **nerdd**
  * set **project name** to **default**
  * set **repository URL** to **https://github.com/molinfo-vienna/nerdd**
  * set **path** to **/**
  * set **cluster URL** to **https://kubernetes.default.svc**
  * set **namespace** to **default**


### Passwords

We use `external-secrets` to generate passwords for all dashboards and external tools in the
cluster. After deploying, the passwords can be retrieved using `kubectl`:

```bash
kubectl get secret prometheus-auth -n monitoring -o jsonpath="{.data.password}" | base64 --decode
kubectl get secret dashboard-auth -n traefik -o jsonpath="{.data.password}" | base64 --decode
kubectl get secret grafana-auth -n monitoring -o jsonpath="{.data.password}" | base64 --decode
kubectl get secret redpanda-auth -n dev -o jsonpath="{.data.password}" | base64 --decode
kubectl get secret -n argocd git-creds -o jsonpath='{.data.githubAppPrivateKey}' | base64 -d
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
multiple git repositories for infrastructure and code. Especially, it enables specifying the order 
of how apps are deployed.

## Uninstall

```sh
argocd app delete nerdd
# confirm that all data will be destroyed
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'
```

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

## Contribute

* Install docker
* Install a variant of kubernetes, e.g. microk8s, minikube, k3s, k3d or kind
* Install tilt
* `tilt up`
* visit localhost:10350 for tilt dashboard
* visit localhost:8443 for frontend application
* visit localhost:8443/api/ for backend api