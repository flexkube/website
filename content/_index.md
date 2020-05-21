# Flexkube

Flexkube is a minimalistic and flexible Kubernetes distribution, providing tools to manage each main Kubernetes component independently, which can be used together to create Kubernetes clusters.

Core part of the project is [libflexkube](https://github.com/flexkube/libflexkube), a Go library providing logic for managing Kubernetes components (e.g. [etcd](https://etcd.io/), [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)) and reference implementation of CLI tool and [Terraform](https://terraform.io) provider.

## Installation and usage

### CLI tool

For more detailed instructions see the user guide.

If you have `go` binary available in your system, you can start using Flexkube for creating Kubernetes certificates just by running the following commands:
```sh
echo 'pki:
  kubernetes: {}
  etcd: {}' > config.yaml
go run github.com/flexkube/libflexkube/cmd/flexkube pki
```

It will create `config.yaml` configuration file consumed by `flexkube` CLI tool and generate the certificates into `state.yaml` file.

### Terraform

If you want to perform the same action using Terraform.

```sh
VERSION=v0.2.2 wget -qO- https://github.com/flexkube/libflexkube/releases/download/$VERSION/terraform-provider-flexkube_$VERSION_linux_amd64.tar.gz | tar zxvf - terraform-provider-flexkube_$VERSION_x4
echo 'resource "flexkube_pki" "pki" {
  etcd {}
  kubernetes {}
}' > main.tf
terraform init && terraform apply
```

After applying, Kubernetes certificates should be available as Terraform resource attributes of `flexkube_pki.pki` resource.

### TODO

- For who is this project suitable for
- For who is this project not really suitable
- Getting started
  - Installation
    - CLI
    - Terraform
  - 