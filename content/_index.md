# Flexkube

Flexkube is a minimalistic and flexible Kubernetes distribution, providing tools to manage each main Kubernetes component independently, which can be used together to create Kubernetes clusters.

Core part of the project is [libflexkube](https://github.com/flexkube/libflexkube), a Go library providing logic for managing Kubernetes components (e.g. [etcd](https://etcd.io/), [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/), [certificates](https://kubernetes.io/docs/setup/best-practices/certificates/)) and reference implementation of CLI tool and [Terraform](https://terraform.io) provider.

## Installation and usage

For quick examples of installation and usage, see the content below. For full documentation, see [Getting started]({{< relref "/documentation/getting-started" >}}).

### CLI tool

If you have `go` binary available in your system, you can start using Flexkube for creating Kubernetes certificates just by running the following commands:
```sh
cat <<EOF > config.yaml
pki:
  kubernetes: {}
  etcd: {}' > config.yaml
EOF
go run github.com/flexkube/libflexkube/cmd/flexkube pki
```

It will create `config.yaml` configuration file which will be consumed by `flexkube` CLI tool, which will then generate the certificates into `state.yaml` file.

### Terraform

If you want to perform the same action using Terraform, execute the following commands:

```sh
cat <<EOF > main.tf
terraform {
  required_providers {
    flexkube = {
      source  = "flexkube/flexkube"
      version = "0.4.0"
    }
  }
}

resource "flexkube_pki" "pki" {
  etcd {}
  kubernetes {}
}
EOF
terraform init && terraform apply
```

After applying, Kubernetes certificates should be available as Terraform resource attributes of `flexkube_pki.pki` resource.

### Next steps

For more detailed instructions of how to use Flexkube, see the user [guides](https://flexkube.github.io/documentation/guides).

## Characteristics

Flexkube project focuses on simplicity and tries to only do minimal amount of steps in order to get Kubernetes cluster running, while keeping the configuration flexible and extensible. It is also a good material for learning how Kubernetes cluster is set up, as each part of the cluster is managed independently and code contains a lot of comments why specific flag/option is needed and what purpose does it serve.

Parts of this project could possibly be used in other Kubernetes distributions or be used as a configuration reference, as setting up Kubernetes components requires many flags and configuration files to be created.

Flexkube do not manage infrastructure for running Kubernetes clusters, it must be provided by the user.

## Features

Here is the short list of Flexkube project features:

- Minimal host requirements - Use SSH connection forwarding for talking directly to the container runtime on remote machines for managing static containers and configuration files.
- Independent management of etcd, kubelets, static control plane and self-hosted components.
- All self-hosted control plane components managed using Helm 3 (e.g CoreDNS).
- 1st class support for Terraform provider for automation.
- No Public DNS or any other public discovery service is required for getting cluster up and running.
- Others:
  - etcd, kubelet and static control plane running as containers.
  - Self-hosted control plane.
  - Supported container runtimes:
    - Docker
  - Configuration via YAML or via Terraform.
  - Deployment using CLI tools or via Terraform.
  - HAProxy for load-balancing and fail-over between Kubernetes API servers.

## Known issues, missing features and limitations

As the project is still in the early stages, here is the list of major existing issues or missing features, which are likely to be implemented in the future:

- gracefully replacing CA certificates (if private key does not change, it should work, but has not been tested)
- no checkpointer for pods/apiserver. If static kube-apiserver container is stopped and node reboots, single node cluster will not come back.

And features, which are not yet implemented:

- network policies for kube-system namespace
- caching port forwarding
- bastion host(s) support for SSH
- parallel deployments across hosts
- removal of configuration files, created data and containers
- automatic shutdown/start of bootstrap control plane

## Contributing

All contributions to this project are welcome. If it does not satisfy your needs, feel free to raise an issue about it or implement the support yourself and create a pull request with the patch, so we can all benefit from it.

If you just want to help the project grow and mature, there are many TODOs spread across the code, which should be addresses sooner or later.

## Status of the project

At the time of writing, this project is in active development state and it is not suitable for production use. Breaking changes might be introduced at any point in time, for both library interface and for existing deployments.
