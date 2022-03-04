# Terraform + Fluxcd

This note describes the usage of Terraform with Fluxcd.

## Concepts

[Fluxcd](https://fluxcd.io/) - GitOps operator for Kubernetes.<br/>
[Terraform](https://www.terraform.io/) - Widespread IAC tool.

Terraform creates a Kubernetes cluster then calls Flux to bootstrap a repository.
Flux generates a set of manifests with CRDs and other Kubernetes objects, including the controllers.
Then it commits these manifests to the specified repository and applies them to the Kubernetes apiserver.

The issue is that the most examples use `main` branch as the target to `flux bootstrap` command. Do you really push your code directly to `main`? Hardly so.
This note covers the case when the `main` branch in you repository is closed for directo commits and can only apply merge requests. Also we assume that you do not want to create a separate repository and there is some code in the existing one.

### Workflow

1. Configure Flux to bootstrap on `new` branch of the existing repository.
2. Rebase `new` branch on `main` branch.
3. Call Terraform to invoke `flux bootstrap`.
4. Add an application manifest at `new` branch.
5. Create a merge request from `new` branch to `main` branch.
6. Merge `new` branch into `main`.

## Tools used

* Gitlab as git repository storage with HTTPS interface.
* GKE as Kubernetes engine.
* [Flux CLI](https://fluxcd.io/docs/cmd/) as a tool for bootstrapping the repository. There is also [terraform provider for fluxcd](https://github.com/fluxcd/terraform-provider-flux), though it does not look like something convenient.

## Code

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
