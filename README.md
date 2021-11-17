# gitops-flux-helm-k8s-sops
### A GitOPS example with Flux, Kubernetes, Git, and sops

The goal of this article is an example of the benefits of the GitOPS tools like FluxCD, EKS, SOPS, KMS.
How Setup a next-gen Kubernetes deployment depicting your current technology.

A GitOPS example with Flux, Kubernetes, Git, and sops

https://www.garanet.net/gitops-example-flux-kubernetes-gitlab-sops/

### The tools used for this example are:

- K8s cluster =  Docker-Desktop (locally) – EKS (desirable).
- GITLAB = Git Enterprise Trial (SaaS) – Enterprise version (desirable).
- FluxCD = Flux bootstrap to orchestrate the deployments.
- LENS / Terminal = IDE to see the cluster’s status/metrics.

For a Local Test, we assume a scenario with two clusters: staging and production.

### Prerequisites

- Kubernetes cluster in this case you can use the Docker-Desktop instead of an AWS EKS.
- Kubectl installed on your machine.
- FluxCD installed on your machine.
- A Git repos with a personal token auth.

### Repository structure

The Git repository contains the following top directories:

- infrastructure contains common infra tools such as ingress controller, storage drivers, etcd and Helm repository definitions.
- microservices contains Helm releases (applications) with a custom configuration per cluster (production/staging)
- clusters contains the Flux configuration per cluster (production/staging)
- secrets contains the encrypted security-config or configmap per cluster (production/staging)

```
|── clusters
|   ├── production
|   └── staging
|── infrastructure
|   ├── base 
|   ├── production
|   └── staging
|── microservices
|   ├── base 
|   ├── production
|   └── staging
|── secrets
|   ├── production 
|   └── staging
```

### The Architecture
After the main GitLab folders structure has been made, we need to create the Kustomization files and the most important one is the helmrepositories folder.

In order to test the podinfo app, it requires an NGINX and a REDIS deployment.

Inside the ./infrastructure/base/ the structure will be:

```
|── helmrepositories
|   ├── kustomization.yaml
|   ├── bitnami.yaml
|   └── podinfo.yaml
|── redis
|   ├── kustomization.yaml
|   ├── namespace.yaml
|   └── redis-release.yaml
|── nginx    
    ├── kustomization.yaml    
    ├── namespace.yaml    
    └── nginx-release.yaml
```
Now we need to define the NGINX and REDIS values for the application or the deployment itself, like persistent volumes, env variables.

Inside the ./infrastructure/production or ./infrastructure/staging the structure will be:
```
|── production
|   ├── kustomization.yaml
|   ├── nginx-values.yaml
|   └── redis-values.yaml
|── staging
|   ├── kustomization.yaml
|   ├── nginx-values.yaml
|   └── redis-values.yaml
```
The microservices

It is the same logic of the infrastructure folder where the base contains the main helm deployment, and inside the folders production and staging the patch values for the helm deployment, in this case for the podinfo app.

Inside the ./microservices/ the structure will be:
```
|── base
|   └── podinfo
|       ├── kustomization.yaml
|       ├── namespace.yaml
|       └── podinfo.yaml
|── production
|   ├── kustomization.yaml
|   └── podinfo-values.yaml
|── staging
    ├── kustomization.yaml
    └── podinfo-values.yaml
```
Bootstrap staging and production

We need to create the latest files before running the Flux bootstrap. These files help flux to retrieve the Kustomization files for each environment (production/staging).

Inside the ./clusters/ the structure will be:
```
|── production
|   ├── microservices.yaml
|   └── infrastructure.yaml
|── staging
    ├── microservices.yaml
    └── infrastructure.yaml
```
Run Flux Bootstrap on production

After cloning the repo into a machine, and exporting the gitlab_token, the flux bootstrap to orchestrate the repository can be run.
```
flux bootstrap gitlab \
    --context=production \
    --owner=GITLAB_USER} \
    --repository=GITLAB_REPO \
    --branch=main \
    --token-auth \
    --path=clusters/production \
    --components-extra=image-reflector-controller,image-automation-controller
```
The bootstrap command commits the manifests for the Flux components in clusters/production/flux-system dir and creates a deploy key with read-only access on Git, so it can pull changes inside the cluster.

Check the cluster

A fast way to check the deployment/cluster is via command line:
```
:~# flux get helmreleases --all-namespaces

or

:~# kubectl get helmreleases -A
```
In the alternative, you can install a Kubernetes IDE, like LENS and then you’ll see the deployments:

The Resources/Metrics or the configuration like Persistent volumes/ Ingresses controller.

From the podinfo ingress controller port, in the browser, it will show the pod is working.
Encryption with SOPS

In this local POC there is not the AWS KMS (with the rotation keys) installed. 
But it can be used locally with the GPG like the following example (POC).

### SOPS installation:

macOS
```
$ brew install sops
```
Windows
```
- Go to the latest release page: https://github.com/mozilla/sops/releases/latest
- Download sops-v3.5.0.exe (or whatever the latest version is)
- Rename the file from sops-v3.5.0.exe to just sops.exe
- Copy the file sops.exe to C:\Windows\System32.
– Alternately, put the file in any directory and set the Path environment variable accordingly
```
GPG Configuration

Install GPG:

– Generate a new pgp key with: gpg –full-generate-key
– List of gpg keys with: gpg –list-keys
– Export private keys with: gpg -o private.key –armor –export-secret-keys email@..
– Create a file .sops.yaml at the project folder (/secrets/production/.sops.yaml).
– Import the pub keys to gitlab settings/CI/CD/Variables. (Add KEY as name and FILE as type).
– Add the new customization to flux-system.

The SOPS file looks like this:
```
creation_rules:
  - path_regex: .*.yaml
    pgp: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    encrypted_regex: ^(data|stringData)$
```
Encrypt & Decrypt files with SOPS:

Encryption with:
```
:# sops -i -e --encrypted-regex '^(data|stringData)$' /secrets/production/wordpress-sc.yaml
```

Decryption (only the pgp allowed or AWS KMS user allowed)
```
:# sops -d /secrets/production/rabbitmq-sc.yaml > decrypted-rabbitmq-sc.yaml
```
Working with Docker image from GIT

K8s allows to manage and add an image scanner policy to get and deploy a new version of it.
Flux v2 uses the Gitops toolkit which contains many controllers and can be configured like:

- Source Controller
- Kustomize Controller
- Helm Controller
- Notification Controller

Ref: https://fluxcd.io/docs/components/

The image-reflector-controller implements the image metadata reflection controller (scans container image repositories and reflects the metadata in Kubernetes resources); this repository implements the image update, automation controller.

This is an example of configuring automation.

Going to use cuttlefacts-app because it’s minimal and easy to follow.
Going to configure where the docker builds image repos should be taken:

Created a GitRepository which will provide access to the git repository within the cluster in:

./infrastructure/base/gitrepositories/app-repo.yaml
```
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: cuttlefacts-repo
  namespace: flux-system
spec:
  url: ssh://git@github.com/squaremo/cuttlefacts-app
  interval: 1m
  secretRef:
    name: cuttlefacts-deploy
```
Created Image repository name and Policy in:

./infrastructure/base/imagerepositories/app-image.yaml
```
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImageRepository
metadata:
  name: app-image
  namespace: flux-system
spec:
  image: cuttlefacts/cuttlefacts-app
---
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImagePolicy
metadata:
  name: app-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: app-image
  policy:
    semver:
      range: 1.0.x
```
Then created the deployment and the services inside:

./microservices/base/cuttlefacts/
./microservices/production/cuttlefacts/

After flux reconcile the kustomization, it will show:
```
NAMESPACE     NAME         READY    MESSAGE                         LAST SCAN               SUSPENDED
flux-system imagerepository/app-image True successful scan, found 3 tags 2021-10-19T11:18:04+02:00 False

NAMESPACE     NAME         READY    MESSAGE                       LATEST IMAGE
flux-system    imagepolicy/app-policy    True     Latest image tag for 'cuttlefacts/cuttlefacts-app' resolved to: 1.0.0    cuttlefacts/cuttlefacts-app:1.0.0

NAMESPACE     NAME               READY    MESSAGE               REVISION            SUSPENDED
flux-system    gitrepository/cuttlefacts-repo
```

References:

GITOPS:

– https://kubernetes.io/ 
- https://fluxcd.io/docs/get-started/
- https://fluxcd.io/docs/cmd/flux_bootstrap_gitlab/
- https://artifacthub.io/
- https://www.indellient.com/blog/what-are-the-benefits-of-gitops/
- https://github.com/fluxcd/kustomize-controller

SOPS:

– https://github.com/mozilla/sops
- https://www.varokas.com/secrets-in-code-with-mozilla-sops/
- https://dev.to/stack-labs/manage-your-secrets-in-git-with-sops-gitlab-ci-2jnd
- https://dev.to/stack-labs/manage-your-secrets-in-git-with-sops-common-operations-118g

BUILDING:

- https://fluxcd.io/docs/use-cases/jenkins/
- https://github.com/kingdonb/jenkins-example-workflow
- https://itnext.io/continuous-delivery-with-gitops-591ff031e8f9

Check also how: https://www.garanet.net/aws-protected-website-cloudfront-lambda/