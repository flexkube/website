# Flexkube

Flexkube is a minimalistic and flexible Kubernetes distribution, providing tools to manage each main Kubernetes component independently, which can be used together to create Kubernetes clusters.

Core part of the project is [libflexkube](https://github.com/flexkube/libflexkube), a Go library providing logic for managing Kubernetes components (e.g. [etcd](https://etcd.io/), [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)) and reference implementation of CLI tool and [Terraform](https://terraform.io) provider.

## Installation and usage

### CLI tool

For more detailed instructions see the user [guides]({{< relref "/documentation/guides" >}}).

If you have `go` binary available in your system, you can start using Flexkube for creating Kubernetes certificates just by running the following commands:
```sh
echo 'pki:
  kubernetes: {}
  etcd: {}' > config.yaml
go run github.com/flexkube/libflexkube/cmd/flexkube pki
```

It will create `config.yaml` configuration file which will be consumed by `flexkube` CLI tool, which will then generate the certificates into `state.yaml` file.

### Terraform

If you want to perform the same action using Terraform, execute the following commands:

```sh
VERSION=v0.3.0 wget -qO- https://github.com/flexkube/libflexkube/releases/download/$VERSION/terraform-provider-flexkube_$VERSION_linux_amd64.tar.gz | tar zxvf - terraform-provider-flexkube_$VERSION_x4
echo 'resource "flexkube_pki" "pki" {
  etcd {}
  kubernetes {}
}' > main.tf
terraform init && terraform apply
```

After applying, Kubernetes certificates should be available as Terraform resource attributes of `flexkube_pki.pki` resource.

### Characteristics

Flexkube project focuses on simplicity and tries to only do minimal amount of steps in order to get Kubernetes cluster running, while keeping the configuration flexible and extensible. It is also a good material for learning how Kubernetes cluster is set up, as each part of the cluster is managed independently and code contains a lot of comments why specific flag/option is needed and what purpose does it serve.

Flexkube do not manage infrastructure for running Kubernetes clusters, it must be provided by the user.