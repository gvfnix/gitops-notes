# Terraform + Fluxcd

This note describes the usage of Terraform with Fluxcd.

## Concepts

[Fluxcd](https://fluxcd.io/) - GitOps operator for Kubernetes.<br/>
[Terraform](https://www.terraform.io/) - Widespread IAC tool.

Terraform creates a Kubernetes cluster then calls Flux to bootstrap a repository.
Flux generates a set of manifests with CRDs and other Kubernetes objects, including the controllers.
Then it commits these manifests to the specified repository and applies them to the Kubernetes apiserver.

The issue is that the most examples use `main` branch as the target to `flux bootstrap` command. Do you really push your code directly to `main`? Hardly so.
This note covers the case when the `main` branch in you repository is protected from direct commits and can only apply merge requests. Also we assume that you do not want to create a separate repository and there is some code in the existing one.

### Workflow

1. Configure Flux to bootstrap on `new` branch of the existing repository.
1. Rebase `new` branch on `main` branch.
1. Add an application manifest at `new` branch.
1. Commit the changes.
1. Wait for Flux Kustomization controller to apply the changes (typically 10-30 seconds).
1. Change file `gotk-sync.yaml` so that `source.toolkit.fluxcd.io/v1beta1/GitRepository[flux-system/flux-system]` had `spec.ref.branch == main`.
1. Create a merge request from `new` branch to `main` branch.
1. Merge `new` branch into `main`.

## Tools used

* Gitlab as git repository storage with HTTPS interface.
* GKE as Kubernetes engine.
* [Flux CLI](https://fluxcd.io/docs/cmd/) as a tool for bootstrapping the repository. There is also [terraform provider for fluxcd](https://github.com/fluxcd/terraform-provider-flux), though it does not look like something convenient.

## Code

### Step 1. Bootstrap Flux on `new` branch.

**cluster.tf**
```terraform
locals {
  project_id      = "gvfnix-1"
  cluster_name    = "gvnix-1"
  network_name    = "gvfnix-1"
  subnetwork_name = "gke-1"
  region          = "us-central-1"
}
data "google_client_config" "default" {}

provider "kubernetes" {
  host                   = "https://${module.gke.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.gke.ca_certificate)
}

module "gke" {
  source                     = "terraform-google-modules/kubernetes-engine/google"
  project_id                 = local.project_id
  name                       = local.cluster_name
  region                     = local.region
  network                    = local.network_name
  subnetwork                 = local.subnetwork_name
  ip_range_pods              = "${local.subnetwork_name}-pods"
  ip_range_services          = "${local.subnetwork_name}-svc"
  http_load_balancing        = false
  horizontal_pod_autoscaling = false
  network_policy             = false
}
```

**fluxcd.tf**
```terraform
locals {
  fluxcd_settings = {
    hostname   = "gitlab.com"
    branch     = "new"
    token      = data.google_secret_manager_secret_version.fluxcd_gitlab_token.secret_data
    owner      = "gvfnix"
    repository = "gitops"
    path       = "clusters/${local.cluster_name}/preprod"
    personal   = true
  }
  flux_bootstrap_command = <<-EOT
  gcloud container clusters get-credentials ${local.cluster_name} --region=${local.region}
  flux bootstrap gitlab --token-auth \
    --hostname=${local.fluxcd_settings.hostname} \
    --owner=${local.fluxcd_settings.owner} --repository=${local.fluxcd_settings.repository} \
    --branch=${local.fluxcd_settings.branch} --path=${local.fluxcd_settings.path} \
    --personal=${local.fluxcd_settings.personal}
  EOT
}

data "google_secret_manager_secret_version" "fluxcd_gitlab_token" {
  secret  = "gitlab-token_api-developer_gvfnix-gitops"
  project = local.project
}

resource "null_resource" "fluxcd_operators" {
  provisioner "local-exec" {
    command = local.flux_bootstrap_command
    environment = {
      GITLAB_TOKEN = local.fluxcd_settings.token
    }
  }
  depends_on = [module.gke]
  triggers = {
    command = sha256(local.flux_bootstrap_command)
    token   = local.fluxcd_settings.token
  }
}
```

When we call `terraform plan` and then `terrraform apply`, it will create the cluster and commit three files to the repository at `clusters/gvfnix-1/preprod/flux-system` directory. These files are:

* `gotk-components.yaml` - CRDs and controllers. This file should hardly ever change.
* `gotk-sync.yaml` - custom resources that specify the target repository. We will change it later.
* `kustomization.yaml`

### Step 2. Rebase `new` to `main`.

```bash
git clone https://gitlab.com/gvfnix/gitops.git
cd gitops
git checkout new
git rebase main
git push -u origin HEAD
```

### Step 3. Add manifests for `sealed-secrets` helm release.
```bash
mkdir -p ./clusters/gvfnix-1/apps/base/sealed-secrets
cat << 'EOF' > ./clusters/gvfnix-1/apps/base/sealed-secrets/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kube-system
resources: [resources.yml]
EOF

cat << 'EOF' > ./clusters/gvfnix-1/apps/base/sealed-secrets/resources.yml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: sealed-secrets-controller
spec:
  interval: 1h
  url: https://bitnami-labs.github.io/sealed-secrets
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sealed-secrets-controller
spec:
  interval: 5m
  chart:
    spec:
      chart: sealed-secrets
      version: "2.1.3"
      sourceRef:
        kind: HelmRepository
        name: sealed-secrets-controller
      interval: 5m
---
EOF

cat << 'EOF' > ./clusters/gvfnix-1/preprod/apps/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/base/sealed-secrets
EOF

git add . && git commit -m "Add sealed-secrets helm release"
```

### Step 3. Change Flux target branch to `main`.

**./clusters/gvfnix-1/preprod/flux-system/gotk-sync.yaml**
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: new # <-- This should be changed to `main`
  secretRef:
    name: flux-system
  url: https://gitlab.com/gvfnix/gitops.git
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/dbops/preprod
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
---
```

```bash
git commit -am "Change fluxcd manifests to pull the changes from 'main' branch"
git push
```

After this step Flux's source controller will be aimed to `main` branch. It will fail to pull the changes as the files you have in `new` branch do not yet exist in `main`.

### Step 4. Merge your `new` branch into `main`.

### Step 5. Change your `fluxcd.tf` file to use `main` branch as a bootstrapped one.
