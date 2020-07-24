# Requirements

This section describes various requirements of Flexkube.

{{< hint warning >}}

It is recommended to deploy Flexkube resources (e.g. etcd, kubelet) into dedicated machine, **not** into local host, as resources will write to some hosts directories like `/etc/kubernetes`, `/var/lib/kubelet` or `/var/lib/etcd` to persist the cluster state across updates. 

{{< /hint >}}

## Summary

Short summary of the requirements for each machine where Kubernetes will be deployed:

- Minimum 2 GB of RAM
- SSH server configured (if deploying to remote machines)
- Internet access
- Docker daemon installed and running

## Hardware requirements

To create Kubernetes cluster using Flexkube, you need a machine with at least 2 GB of RAM for controller node and at least 1 GB of RAM for worker nodes.

## Connectivity

### Containers registry

Machines which will be part of the cluster must have access to container registry from where the cluster component images will be pulled. By default public registries are used, so machines **must have internet access**.

If you re-configure the cluster to use images from private repository, internet access should not be required.

### SSH

For deploying on remote machines, Flexkube use SSH tunnels to talk to container runtime on remote machine, so make sure SSH daemon is configured on them and is accessible from the host you will be deploying.

If you deploy only on local machine **or your container runtime on remote machines is directly reachable over network** SSH is not required.

User used for SSH connection should have permissions to talk to local [container runtime](#container-runtime).

### Network

It is recommended, that all machines which are part of the cluster are connected using private network, to avoid exposing your cluster components to the internet.

## Container runtime

Flexkube runs all of Kubernetes controlplane components as containers, so container runtime must be installed and configured on the machines before deploying.

At the moment only **Docker** runtime is supported. In the future, support for more container runtime might be added.

## Creating virtual machines locally for testing.

If you don't have a suitable machine available for testing, see [Creating virtual machines for testing]({{< relref "/documentation/getting-started/requirements/creating-virtual-machines-for-testing" >}}) page, which explains how to create one on your local machine.