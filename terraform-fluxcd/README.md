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

... TODO ...
