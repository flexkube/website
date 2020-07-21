# Creating single-node etcd cluster on local machine using "flexkube" CLI

This guide describes how to create single member etcd cluster using `flexkube` CLI. It will explain cluster creation process step by step to explain the configuration and provide some insights.

For fully automated creation, see [Creating single-member etcd cluster on local machine using Terraform]({{< relref "/documentation/guides/etcd/creating-single-member-cluster-on-local-machine-using-terraform" >}}).

## Requirements

For this guide, it is required to have one Linux machine, with Docker daemon installed and running.

It is recommended that machine has at least 1 GB of RAM and is a fresh machine, as in tutorial the tools will write to directories like `/etc/kubernetes` or `/var/lib/etcd` without notice.

The Docker version should be 18.06+. You can follow [Docker documentation](https://docs.docker.com/get-docker/) to see how to install Docker on your machine.

Network interfaces setup is not important, however having a private IP address is recommended from security perspective.

{{< expand "I don't have such machine." >}}

If you don't have such machine available, you can create it locally, using [VirtualBox](https://www.virtualbox.org/) and [Vagrant](https://www.vagrantup.com/). Make sure you have both tools installed by following respective guides:

- [Installing VirtualBox](https://www.virtualbox.org/manual/ch02.html)
- [Installing Vagrant](https://www.vagrantup.com/docs/installation/)

Once done, create file named `Vagrantfile` with following content:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box       = "flatcar-stable"
  config.vm.box_url   = "https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_vagrant.box"
  config.ssh.username = 'core'
  config.vm.provider :virtualbox do |v|
    v.memory = 1024
  end
end
```

Then, run the following commands to create and connect to the machine:

```sh
vagrant up && vagrant ssh
```

{{< /expand >}}

## Preparation

Before we start creating a cluster, we need to gather some information and download required binaries.

Log in into the machine where you want to deploy etcd before proceeding.

### IP address for etcd member

IP addresses of members must be known ahead of cluster creation time.

You can find available IP addresses on your machine using e.g. `ifconfig` tool.

You can try getting the IP address automatically using the following command:

```sh
export IP=$(ip addr show dev $(ip r | grep default | tr ' ' \\n | grep -A1 dev | tail -n1) | grep 'inet ' | awk '{print $2}' | cut -d/ -f1); echo $IP
```

On VirtualBox, we can use `10.0.2.15` IP.

Save the IP address for future use using the following command:

```sh
export IP=10.0.2.15
```

### Downloading `flexkube` binary

Once logged in, execute the following command to download `flexkube` CLI binary into working directory. This is the binary, which will be used to create a cluster components.

```sh
export VERSION=v0.3.0
wget -O- https://github.com/flexkube/libflexkube/releases/download/${VERSION}/flexkube_${VERSION}_linux_amd64.tar.gz | tar zxvf -
```

### Downloading `etcdctl` binary (optional)

To test cluster functionality, you can download `etcdctl` binary, however, this is optional. Also, if you use Flatcar Container Linux, the binary should be available on the system already.

You can download it using the following command:

```sh
export ETCD_VER=v3.4.9
wget https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -O- | tar zxvf - etcd-${ETCD_VER}-linux-amd64/etcdctl && mv etcd-${ETCD_VER}-linux-amd64/etcdctl ./ && rm
dir etcd-${ETCD_VER}-linux-amd64
```

### Make downloaded binaries available in $PATH

For compatibility with rest of the tutorial, you should make sure that downloaded binaries are in one of the directories in the $PATH environment variable.

You can also add working directory to the $PATH using the following command:

```sh
export PATH="$(pwd):${PATH}"
```

### Checking Docker availability

To avoid runtime issues while running `flexkube`, run the following command to ensure, that Docker is running and is accessible on your machine:

```sh
docker ps
```

## Creating the cluster

Now that you have all required binaries and information, we can start creating the cluster.

### Creating certificates

First step to create a cluster is to generate all certificates required by etcd. For that, we will use Flexkube [PKI resource]({{< relref "/documentation/resources/pki" >}}).

Before we create the certificates, we need to provide some configuration to tell PKI resource to create etcd certificates, as by default it only creates Root CA certificate.

For this guide, you can create configuration using the following command:

```yaml
cat <<EOF | sed '/^$/d' > config.yaml
pki:
  etcd:
    peers:
      member1: ${IP}
EOF
```

{{< hint info >}}

See [PKI configuration reference]({{< relref "/documentation/reference/cli/configuration/pki" >}}) to see all available configuration options.

{{< /hint >}}

In the following example, we use `member1` as a etcd member name. There is no strict convention about the names. [etcd documentation](https://etcd.io/docs/v3.4.0/op-guide/configuration/#--name) suggests to use the `hostname` or `machine-id`, which might be a good choice, if you plan to run only one member on a single machine. Please note, that changing the member name here must also be reflected in next steps of the tutorial.

Once created, run the following command to generate the certificates:

```sh
flexkube pki
```

If everything succeeded, you should find many certificates in newly created `state.yaml` file.

{{< hint info >}}

You can inspect `state.yaml` file using the following command:

```sh
less state.yaml
```

In there, you should find etcd CA certificate and private key, peer and server certificates and private keys for all members we defined (in this tutorial only `member1`) and root CA certificate with private key.

The certificates properties are generated with accordance with [Kubernetes PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/).

{{< /hint >}}

### Creating etcd cluster

With certificates ready, we can now create etcd cluster using [etcd]({{< relref "/documentation/resources/etcd" >}}) resource.

To create etcd cluster, we need to configure it's members in `config.yaml` file. This can be done using the following command:

```yaml
cat <<EOF >> config.yaml

etcd:
  members:
    member1:
      peerAddress: ${IP}
EOF
```

{{< hint info >}}

See [etcd configuration reference]({{< relref "/documentation/reference/cli/configuration/etcd" >}}) to see all available configuration options.

{{< /hint >}}

Now, you can run the following command to create etcd cluster:

```sh
flexkube etcd
```

When you execute this command, it will print you the list of the containers which will be created and ask you to confirm it.

You can also run it with `--yes` flag, to skip the confirmation.

After confirmation, `flexkube` binary will by default talk to Docker runtime over UNIX socket on local host and create desired containers.

{{< hint info >}}

On consecutive runs, `flexkube` will first check the state of the created containers and then, it there is any update pending (e.g. container image update), it will again show you the diff which will be applied and ask you for confirmation.

You can also run it with `--noop` to only see if there are some updates pending.

{{< /hint >}}

Once finished, if you run `docker ps`, you should see etcd container running.

### Inspecting state.yaml file (optional)

With etcd cluster created, state about running containers will be stored in `state.yaml` file. Storing state is needed to calculate configuration updates in the future runs and also to allow cleaning up created containers.

You can have a look into state file using the following command:

```sh
less state.yaml
```

In there, you can find list of all containers which has been created, their configuration files, flags and on which host and using which container runtime they has been created. This is useful if you want to inspect the configuration of created containers.

## Verifying cluster functionality

To verify that the cluster is healthy, we will use etcd member certificates itself and previously downloaded `etcdctl` binary.

First, we need to prepare the environment variables used by `etcdctl` to define how to authenticate to cluster. This can be done using the following commands:

```sh
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/etcd/peer.crt
export ETCDCTL_KEY=/etc/kubernetes/etcd/peer.key
export ETCDCTL_ENDPOINTS="https://10.0.2.15:2379"
```

Now, we check if all endpoints are healthy, using this command:

```
sudo -E etcdctl endpoint health
```

{{< hint info >}}

We use `sudo`, as created certificate files are only readable by `root` user. If you are using `root` user already or you don't want to use `sudo`, you can extract client certificates from `state.yaml` file.

{{< /hint >}}

## What's next

With cluster running, you can now start using it, e.g. to deploy Kubernetes cluster. To do that using Flexkube and Terraform, you can follow [Creating single node Kubernetes cluster on local machine using Terraform]({{< relref "/documentation/guides/kubernetes/creating-single-node-cluster-on-local-machine-using-terraform" >}}).

To clean up created resources, see the section below.

## Cleaning up

To clean up the host, first, rename or remove `config.yaml` file, so CLI will be able to clean up the resources. For example, execute:

```sh
mv config.yaml config.yaml.old
```

Now you can remove all containers managed by `flexkube` using following commands:

```sh
flexkube etcd
```

Finally, following directories can be removed as well:

```sh
sudo rm -rf /etc/kubernetes/ /var/lib/etcd/
```
