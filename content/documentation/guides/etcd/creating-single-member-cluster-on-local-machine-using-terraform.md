# Creating single-member cluster on local machine using Terraform

This guide describes how to create single member etcd cluster using Terraform and Flexkube provider. The process is very simple and requires just a few steps.

For more detailed guide, see [Creating single member etcd cluster on local machine using flexkube CLI]({{< relref "/documentation/guides/etcd/creating-single-member-cluster-on-local-machine-using-flexkube-cli" >}}).

## Requirements

For this guide, it is required to have one Linux machine, with Docker daemon installed and running.

It is recommended that machine has at least 1 GB of RAM and is a fresh machine, as in tutorial the tools will write to directories like `/var/lib/etcd` or `/etc/kubernetes` without notice.

The Docker version should be 18.06+. You can follow [Docker documentation](https://docs.docker.com/get-docker/) to see how to install Docker on your machine.

Network interfaces setup is not important, however having a private IP address is recommended from security perspective.

If you don't have such machine, visit [Creating virtual machines for testing]({{< relref "/documentation/getting-started/requirements/creating-virtual-machines-for-testing#single-node" >}}) to see how to create one locally.

## Preparation

Before we start creating a cluster, we need to gather some information and download required binaries.

Log in into the machine where you want to deploy etcd before proceeding.

### IP address for etcd member

IP addresses of members must be known ahead of cluster creation time.

You can find available IP addresses on your machine using e.g. `ifconfig` tool.

You can try getting the IP address automatically using the following command:

```sh
export TF_VAR_ip=$(ip addr show dev $(ip r | grep default | tr ' ' \\n | grep -A1 dev | tail -n1) | grep 'inet ' | awk '{print $2}' | cut -d/ -f1); echo $TF_VAR_ip
```

On VirtualBox, we can use `10.0.2.15` IP.

Save the IP address for future use using the following command:

```sh
export TF_VAR_ip=10.0.2.15
```

### Downloading `terraform` binary

For this guide, you must have `terraform` binary available. You can download it using the following command:

```sh
export TERRAFORM_VERSION=0.13.1
wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip && \
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip && \
rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip
```

### Downloading `etcdctl` binary (optional)

To test cluster functionality, you can download `etcdctl` binary, however, this is optional. Also, if you use Flatcar Container Linux, the binary should be available on the system already.

You can download it using the following command:

```sh
export ETCD_VERSION=v3.4.16
wget https://storage.googleapis.com/etcd/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz -O- | tar zxvf - etcd-${ETCD_VERSION}-linux-amd64/etcdctl && mv etcd-${ETCD_VERSION}-linux-amd64/etcdctl ./ && rmdir etcd-${ETCD_VERSION}-linux-amd64
```

### Make downloaded binaries available in $PATH

For compatibility with rest of the tutorial, you should make sure that downloaded binaries are in one of the directories in the $PATH environment variable.

You can also add working directory to the $PATH using the following command:

```sh
export PATH="$(pwd):${PATH}"
```

## Creating the cluster

Now that you have all required binaries and information, we can start creating the cluster.

Create `main.tf` file with the following content:

```tf
terraform {
  required_providers {
    flexkube = {
      source  = "flexkube/flexkube"
      version = "0.5.1"
    }
    local = {
      source  = "hashicorp/local"
      version = "1.4.0"
    }
  }

  required_version = ">= 0.15"
}

variable "ip" {}

variable "name" {
  default = "member01"
}

resource "flexkube_pki" "pki" {
  etcd {
    peers = {
      "${var.name}" = var.ip
    }

    servers = {
      "${var.name}" = var.ip
    }

    client_cns = ["root"]
  }
}

resource "flexkube_etcd_cluster" "etcd" {
  pki_yaml = flexkube_pki.pki.state_yaml

  member {
    name           = var.name
    peer_address   = var.ip
    server_address = var.ip
  }
}

locals {
  ca_cert = "./ca.pem"
  cert    = "./client.pem"
  key     = "./client.key"
}

resource "local_file" "etcd_ca_certificate" {
  content  = flexkube_pki.pki.etcd[0].ca[0].x509_certificate
  filename = local.ca_cert
}

resource "local_file" "etcd_root_user_certificate" {
  content  = flexkube_pki.pki.etcd[0].client_certificates[index(flexkube_pki.pki.etcd[0].client_cns, "root")].x509_certificate
  filename = local.cert
}

resource "local_file" "etcd_root_user_private_key" {
  sensitive_content = flexkube_pki.pki.etcd[0].client_certificates[index(flexkube_pki.pki.etcd[0].client_cns, "root")].private_key
  filename          = local.key
}

resource "local_file" "etcd_environment" {
  filename = "./etcd.env"
  content  = <<EOF
#!/bin/bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=${abspath(local.ca_cert)}
export ETCDCTL_CERT=${abspath(local.cert)}
export ETCDCTL_KEY=${abspath(local.key)}
export ETCDCTL_ENDPOINTS="https://${var.ip}:2379"
EOF

  depends_on = [
    flexkube_etcd_cluster.etcd,
  ]
}
```

Now, to create the cluster run following commands:

```sh
terraform init && terraform apply
```

Terraform should pick up the IP address automatically, if you exported it to `TF_VAR_ip` environment variable.

If everything went successfully, you should see now running etcd container, when you execute `docker ps`.

## Verifying cluster functionality

Now that the cluster is running, we can verify that it is functional.

### Inspect created files

After creating the cluster, you can find following files in the working directory, created by Terraform:

- `ca.pem` containing etcd CA X.509 certificate in PEM format.
- `client.pem` containing etcd client X.509 certificate in PEM format, with `root` Common Name.
- `client.key` RSA private key in PEM format for certificate in `client.pem`file.
- `etcd.env` containing environment variables needed for `etcdctl`.

Certificates and private key files are required to access the cluster. The `etcd.env` file is just a helper file for this tutorial.

The files can also be safely removed, as all the certificates are stored in Terraform state anyway.

### Using `etcdctl`

`etcdctl` can be used to verify that the cluster is functional and to perform some basic operations as well as administrative tasks.

To be able to use it, it is recommended to set environment variables, pointing to the certificates and cluster members, so they don't have to be repeated for each command.

With this guide, you get `etcd.env` helper file created, from which you can load the environment variables, using following command:

```sh
source etcd.env
```

Now `etcdctl` is ready to use.

To check if cluster is healthy, execute the following command:

```sh
etcdctl endpoint health
```

## What's next

With cluster running, you can now start using it, e.g. to deploy Kubernetes cluster. To do that using Flexkube and Terraform, you can follow [Creating single node Kubernetes cluster on local machine using Terraform]({{< relref "/documentation/guides/kubernetes/creating-single-node-cluster-on-local-machine-using-terraform" >}}).

To clean up created resources, see the section below.

## Cleaning up

First step of removing the cluster is running Terraform, to remove all containers. To perform that, run this command:

```sh
terraform destroy
```

Once finished, you can remove the directories created by the cluster, using the following command:

```sh
sudo rm -rf /var/lib/etcd/ /etc/kubernetes/
```

