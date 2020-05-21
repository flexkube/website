# Requirements

This section describes various requirements of Flexkube.

{{< hint warning >}}

It is recommended to deploy Flexkube resources (e.g. etcd, kubelet) into dedicated machine, **not** into local host, as resources will write to some hosts locations like `/etc/kubernetes`, `/var/lib/kubelet` or `/var/lib/etcd` to persist the cluster state across updates. See TODO section to learn how to create VM for testing.

{{< /hint >}}

## Summary

Short summary of the requirements for each machine where Kubernetes will be deployed:

- Minimum 2 GB of RAM
- SSH server configured (if deploying to remote machines)
- Internet access
- `docker` daemon installed and running

## Hardware requirements

To create Kubernetes cluster using Flexkube, you need a machine with at least 2 GB of RAM for controller node and at least 1 GB of RAM for worker nodes.

## Connectivity

### Containers registry

Machines which will be part of the cluster must have access to container registry from where the cluster component images will be pulled. By default public registries are used, so machines **must have internet access**.

If you re-configure the cluster to use images from private repository, internet access should not be required.

### SSH

For deploying on remote machines, Flexkube use SSH tunnels to talk to container runtime on remote machine, so make sure SSH daemon is configured on them and is accessible from the host you will be deploying.



If you deploy only on local machine, SSH is not required.

### Network

It is recommended, that all machines which are part of the cluster are connected using private network, to avoid exposing your cluster components to the internet.

## Container runtime

Flexkube runs all of Kubernetes controlplane components as containers, so container runtime must be installed and configured on the machines before deploying.

At the moment only **Docker** runtime is supported. In the future, support for more container runtime might be added.