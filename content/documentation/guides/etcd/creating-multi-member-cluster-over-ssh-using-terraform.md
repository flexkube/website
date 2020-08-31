# Creating multi-member cluster over SSH using Terraform

This guide describes how to create multi member etcd cluster using Terraform and Flexkube provider. The process is very simple and requires just a few steps. If you have at least 3 members, your cluster will be able to tolerate loss on one member, so it will be highly available.

## Requirements

For this guide, it is required to have at least 2 Linux machines, with Docker daemon installed and running.

It is recommended that machines has at least 1 GB of RAM and are fresh machines, as in tutorial the tools will write to directories like `/var/lib/etcd` or `/etc/kubernetes` without notice.

The Docker version should be 18.06+. You can follow [Docker documentation](https://docs.docker.com/get-docker/) to see how to install Docker on your machine.

Network interfaces setup is not important, however having a private IP address is recommended from security perspective.

The machines must be able to communicate with each other.

If you don't have such machines, visit [Creating virtual machines for testing]({{< relref "/documentation/getting-started/requirements/creating-virtual-machines-for-testing#multiple-nodes" >}}) to see how to create them locally.

## Preparation

Before we start creating a cluster, we need to gather some information and download required binaries.

### IP addresses for etcd members and SSH

IP addresses of members must be known ahead of cluster creation time.

You can find available IP addresses on your machines using e.g. `ifconfig` tool.

You can try getting the IP address automatically using the following command:

```sh
ip addr show dev $(ip r | grep default | tr ' ' \\n | grep -A1 dev | tail -n1) | grep 'inet ' | awk '{print $2}' | cut -d/ -f1
```

Save the IP addresses of your machines, as they will be needed later on for configuration.

{{< hint warning >}}

If you plan to use different IP addresses for connecting over SSH to your machines and different for members to communicate, note both of them.

{{< /hint >}}

### Downloading `terraform` binary

For this guide, you must have `terraform` binary available. You can download it using the following command:

```sh
export VERSION=0.13.1
wget https://releases.hashicorp.com/terraform/${VERSION}/terraform_${VERSION}_linux_amd64.zip && \
unzip terraform_${VERSION}_linux_amd64.zip && \
rm terraform_${VERSION}_linux_amd64.zip
```

### Downloading `etcdctl` binary (optional)

To test cluster functionality, you can download `etcdctl` binary, however, this is optional. Also, if you use Flatcar Container Linux, the binary should be available on the system already.

You can download it using the following command:

```sh
export ETCD_VER=v3.4.13
wget https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -O- | tar zxvf - etcd-${ETCD_VER}-linux-amd64/etcdctl && mv etcd-${ETCD_VER}-linux-amd64/etcdctl ./ && rmdir etcd-${ETCD_VER}-linux-amd64
```

### Make downloaded binaries available in $PATH

For compatibility with rest of the tutorial, you should make sure that downloaded binaries are in one of the directories in the $PATH environment variable.

You can also add working directory to the $PATH using the following command:

```sh
export PATH="$(pwd):${PATH}"
```

## Creating the cluster

Now that you have all required binaries and information, we can start creating the cluster.

### Terraform configuration

First, create `main.tf` file with the following content:

```tf
terraform {
  required_providers {
    flexkube = {
      source  = "flexkube/flexkube"
      version = "0.4.0"
    }
    local = {
      source  = "hashicorp/local"
      version = "1.4.0"
    }
  }

  required_version = ">= 0.13"
}

variable "members" {
  type = map(object({
    peer_address = string
    ssh_address  = string
  }))
}

variable "ssh_user" {
  default = ""
}

variable "ssh_password" {
  default = ""
}

variable "ssh_private_key" {
  default = ""
}

resource "flexkube_pki" "pki" {
  etcd {
    peers = { for name, member in var.members : name => member.peer_address }

    servers = { for name, member in var.members : name => member.peer_address }

    client_cns = ["root"]
  }
}

resource "flexkube_etcd_cluster" "etcd" {
  pki_yaml = flexkube_pki.pki.state_yaml

  dynamic "member" {
    for_each = var.members

    content {
      name           = member.key
      peer_address   = member.value.peer_address
      server_address = member.value.peer_address

      host {
        ssh {
          user        = var.ssh_user
          password    = var.ssh_password
          private_key = var.ssh_private_key
          address     = member.value.ssh_address
        }
      }
    }
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
export ETCDCTL_ENDPOINTS="${join(",", formatlist("https://%s:2379", [for name, member in var.members : member.peer_address]))}"
EOF

  depends_on = [
    flexkube_etcd_cluster.etcd,
  ]
}
```

### Terraform values

Next, create file named `values.auto.tfvars`, which will store the values required by the Terraform configuration. The file should look like following:

```tf
members = {
  "member1" = {
    peer_address = "192.168.52.10"
    ssh_address  = "192.168.52.10"
  },
  "member2" = {
    peer_address = "192.168.52.11"
    ssh_address  = "192.168.52.11"
  },
}

ssh_user        = "core"
ssh_password    = ""
ssh_port        = 22
ssh_private_key = <<EOF
EOF
```

First, it has defined map of members, where they key is the member name and then each member has peer address and SSH address defined. `peer_address` will be used for etcd and `ssh_address` will be used to SSH into the machines.

Next, make sure that SSH settings are correct. If the SSH key, which is authorized to log in into the machines, is loaded in your `ssh-agent`, you don't need to specify any credentials. Flexkube will automatically pick it up and use it. If not, you can specify content of private key in `ssh_private_key` field or use password authentication using `ssh_password` field.

Using bastion host is currently not supported, though it will be in the future.

### Running Terraform

Now, to create the cluster run following commands:

```sh
terraform init && terraform apply
```

If everything went successfully, you should see now running etcd container, when you execute `docker ps` on the machines.

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

With cluster running, you can now start using it, e.g. to deploy Kubernetes cluster. To do that using Flexkube and Terraform, you can follow [Creating multi node Kubernetes cluster using Terraform]({{< relref "/documentation/guides/kubernetes/creating-multi-node-cluster-using-terraform" >}}).

To clean up created resources, see the section below.

## Cleaning up

First step of removing the cluster is running Terraform, to remove all containers. To perform that, run this command:

```sh
terraform destroy
```

Once finished, you can remove the directories created by the cluster, using the following command **on the machines**:

```sh
sudo rm -rf /var/lib/etcd/ /etc/kubernetes/
```
