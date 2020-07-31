# Creating single-node cluster on local machine using "flexkube" CLI

This guide describes how to create single node Kubernetes cluster using `flexkube` CLI. It will explain cluster creation process step by step to explain the configuration and provide some insights.

For fully automated creation, see [Creating single-node Kubernetes cluster on local machine using Terraform]({{< relref "/documentation/guides/kubernetes/creating-single-node-cluster-on-local-machine-using-terraform" >}}).

## Requirements

For this guide, it is required to have one Linux machine, with Docker daemon installed and running.

It is recommended that machine has at least 2 GB of RAM and is a fresh machine, as in tutorial the tools will write to directories like `/etc/kubernetes` or `/var/lib/kubelet` without notice.

The Docker version should be 18.06+.

Network interfaces setup is not important, however having a private IP address is recommended from security perspective.

If you don't have such machine, visit [Creating virtual machines for testing]({{< relref "/documentation/getting-started/requirements/creating-virtual-machines-for-testing#single-node" >}}) to see how to create one locally.

## Preparation

Before we start creating a cluster, we need to gather some information and download required binaries.

Log in into the machine where you want to deploy Kubernetes before proceeding.

### IP address for deployment

To configure cluster components, you need to provide the IP address, which will be used by the cluster. You can find available IP addresses using e.g. `ifconfig` command.

You can try getting the IP address automatically using the following command:

```sh
export IP=$(ip addr show dev $(ip r | grep default | tr ' ' \\n | grep -A1 dev | tail -n1) | grep 'inet ' | awk '{print $2}' | cut -d/ -f1); echo $IP
```

On VirtualBox, we can use `10.0.2.15` IP.

Save the IP address for future use using the following command:

```sh
export IP=10.0.2.15
```

### Selecting service CIDR and pod CIDR

Kubernetes requires 2 network CIDRs to operate, one from each pod will receive the IP address and one for `Service` objects with type `ClusterIP`. While selecting the CIDRs, make sure they don't overlap with each other and other networks your machine is connected to.

Once decided on CIDRs, we should also save 2 special IP addresses:

- `kubernetes` Service - This IP address will be used by pods which talk to Kubernetes API. It must be included in `kube-apiserver` server certificate IP addresses list. This must be first address of Service CIDR. So if your service CIDR is `11.0.0.0/24`, it should be `11.0.0.1`.
- DNS Service - This IP address will be used by cluster's DNS service. This IP is usually 10th address of Service CIDR. So if your service CIDR is `11.0.0.0/24`, it should be `11.0.0.10`.

With all this information gathered, you command like this to save this information for later use:

```sh
export POD_CIDR=10.0.0.0/24
export SERVICE_CIDR=11.0.0.0/24
export KUBERNETES_SERVICE_IP=11.0.0.1
export DNS_SERVICE_IP=11.0.0.10
```

### Downloading `flexkube` binary

Once logged in, execute the following command to download `flexkube` CLI binary into working directory. This is the binary, which will be used to create a cluster components.

```sh
export VERSION=v0.3.1
wget -O- https://github.com/flexkube/libflexkube/releases/download/${VERSION}/flexkube_${VERSION}_linux_amd64.tar.gz | tar zxvf -
```

### Downloading `kubectl` binary

To verify that cluster is operational it is recommended to have `kubectl` binary available. You can install it using the following command:

```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl && chmod +x kubectl
```

### Downloading `helm` binary

Parts of cluster provisioning is done using [Helm](https://helm.sh/) 3 binary, when deploying the cluster using the `flexkube` CLI. You can install it using the following command:

```sh
wget -O- https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz | tar -zxvf - linux-amd64/helm && mv linux-amd64/helm ./ && rmdir linux-amd64
```

### Make downloaded binaries available in $PATH

For compatibility with rest of the tutorial, you should make sure that downloaded binaries are in one of the directories in the $PATH environment variable.

You can also add working directory to the $PATH using the following command:

```sh
export PATH="$(pwd):${PATH}"
```

## Creating the cluster

Now that you have all required binaries and information, we can start creating the cluster.

### Creating certificates

First step to create a cluster is to generate all certificates required by Kubernetes. As this is not a trivial task to create and manage those certificates, Flexkube provides [PKI resource]({{< relref "/documentation/resources/pki" >}}), which does exactly that.

Before we create the certificates, we need to provide some configuration to tell PKI resource to create for you both etcd and Kubernetes certificates, as by default it only creates Root CA certificate.

For this guide, you can create configuration using the following command:

```yaml
cat <<EOF | sed '/^$/d' > config.yaml
pki:
  etcd:
    clientCNs:
    - kube-apiserver
    peers:
      testing: ${IP}
  kubernetes:
    kubeAPIServer:
      serverIPs:
      - ${IP}
      - ${KUBERNETES_SERVICE_IP}
EOF
```

{{< hint info >}}

See [PKI configuration reference]({{< relref "/documentation/reference/cli/configuration/pki" >}}) to see all available configuration options.

{{< /hint >}}

Once created, run the following command to generate the certificates:

```sh
flexkube pki
```

If everything succeeded, you should find many certificates in newly created `state.yaml` file.

### Creating etcd cluster

Before we start Kubernetes containers, we need etcd cluster. Flexkube provides [etcd]({{< relref "/documentation/resources/etcd" >}}) resource to manage such clusters.

To create etcd cluster, we need to configure it's members in `config.yaml` file. This can be done using the following command:

```yaml
cat <<EOF >> config.yaml

etcd:
  members:
    testing:
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

Once finished, you should see etcd container running, if you run `docker ps`.

### Creating static Kubernetes controlplane

With etcd running, you can now create static Kubernetes controlplane. Static, as Flexkube recommends to run Kubernetes controlplane *self-hosted*, so managed using Kubernetes itself. However, before this can be done, temporary, or *static* controlplane is needed. And this is exactly what [Controlplane]({{< relref "/documentation/resources/controlplane" >}}) resource provides.

You can configure it by running the following command:

```yaml
cat <<EOF >> config.yaml

controlplane:
  apiServerAddress: ${IP}
  apiServerPort: 6443
  kubeAPIServer:
    serviceCIDR: ${SERVICE_CIDR}
    etcdServers:
    - https://${IP}:2379
  kubeControllerManager:
    flexVolumePluginDir: /var/lib/kubelet/volumeplugins
EOF
```

{{< hint info >}}

See [Controlplane configuration reference]({{< relref "/documentation/reference/cli/configuration/controlplane" >}}) to see all available configuration options.

{{< /hint >}}

Now, you can create Kubernetes controlplane using the following command:

```sh
flexkube controlplane
```

Execution can take a while, as Kubernetes docker images must be now pulled.

Once finished, you should see 3 new containers running when you run `docker ps`.

### Getting kubeconfig file

Even though the cluster has no objects or deployments yet, you should be able to access it already. For that, you need `kubeconfig` file. `flexkube` CLI provides `flexkube kubeconfig` command, which will read information about the cluster from configuration and state files and print it to you.

To generate `kubeconfig` file, run the following command:

```sh
flexkube kubeconfig | grep -v "Trying to read" > kubeconfig
```

`kubeconfig` file should be created.

Now, you need to configure Kubernetes clients to use this file. This can be done using the following command:

```sh
export KUBECONFIG=$(pwd)/kubeconfig
```

You can now run `kubectl version` to verify, that the cluster is accessible.

## Adding nodes to the cluster

Having a cluster without nodes is not very useful. This section describes how to add nodes to your cluster.

### Creating TLS bootstrapping RBAC rules and bootstrap tokens

Flexkube requires [TLS bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping) process to be used while adding new nodes to the cluster. To enable that, extra RBAC rules must be created before nodes tries to join the cluster.

This step is handled by [tls-bootstrapping]({{< relref "/documentation/helm-charts/maintained/tls-bootstrapping" >}}) helm chart, which creates RBAC rules and allows to create bootstrap tokens.

First, we need to generate bootstrap token, which will be used in next steps. You can do it by running the following commands:

```sh
export TOKEN_ID=$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 6 | head -n 1)
export TOKEN_SECRET=$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 16 | head -n 1)
```

Then, install the chart to create RBAC rules and bootstrap token, by running this command:

```yaml
helm upgrade --install -n kube-system tls-bootstrapping flexkube/tls-bootstrapping --set tokens[0].token-id=$TOKEN_ID --set tokens[0].token-secret=$TOKEN_SECRET
```

### Creating kubelet pool

With Flexkube, kubelets are managed in pools by [Kubelet Pool]({{< relref "/documentation/resources/kubelet-pool" >}}) resource. This allows to group them to share the configuration. Usually clusters have one group called `controllers` which runs controlplane components and one or more *worker* pools, which might characterize with e.g. different hardware.

For this tutorial, we will just create single pool `default`.

You can configure this pool by running the following command:

```yaml
cat <<EOF >> config.yaml

kubeletPools:
  default:
    bootstrapConfig:
      token: ${TOKEN_ID}.${TOKEN_SECRET}
      server: ${IP}:6443
    adminConfig:
      server: ${IP}:6443
    privilegedLabels:
      node-role.kubernetes.io/master: ""
    volumePluginDir: /var/lib/kubelet/volumeplugins
    kubelets:
    - name: testing
      address: ${IP}
EOF
```

{{< hint info >}}

See [Kubelet pool configuration reference]({{< relref "/documentation/reference/cli/configuration/kubelet-pool" >}}) to see all available configuration options.

{{< /hint >}}

Now, to create `default` pool, run the following command:

```sh
flexkube kubelet-pool default
```

Once finished, you should see that node `testing ` has been added to the cluster by running `kubectl get nodes`.

## Installing CNI, CoreDNS and other packages

Now that you have cluster running with nodes, you need to install some extra packages to make the cluster fully functional.

### Adding helm repositories

Before proceeding, make sure you have `stable` and `flexkube` Helm repositories configured, as it is the recommended source for installing the charts mentioned in next sections. You can add required repositories by running the following commands:

```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add flexkube https://flexkube.github.io/charts/
```

### Installing kube-proxy

`kube-proxy` is not required for bare Kubernetes cluster, so it can be fully managed using Kubernetes itself.

`kube-proxy` handles load balancing traffic to service CIDR in the cluster.

To install it, run the following command:

```sh
helm upgrade --install -n kube-system kube-proxy flexkube/kube-proxy --set "podCIDR=${POD_CIDR}" --set apiServers="{${IP}:6443}"
```

### Installing Calico chart as CNI plugin

While not necessarily required for this guide, as we only run one node, it is recommended to install some CNI plugin on the cluster, as without that, kubelet will stay in `NotReady` state.

Flexkube recommends using [Calico](https://www.projectcalico.org/) as a CNI plugin, as it works on variety of platforms and provides both IPAM and [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) implementation. Flexkube also provides [calico helm chart]({{< relref "/documentation/helm-charts/maintained/calico" >}}), so Calico installation can be easily configured and managed.

To install it, run the following command:

```sh
helm upgrade --install -n kube-system calico flexkube/calico --set flexVolumePluginDir=/var/lib/kubelet/volumeplugins --set podCIDR=$POD_CIDR
```

{{< hint info >}}

We specify `flexVolumePluginDir`, as default path is on `/usr` partition, which is read-only in Flatcar Container Linux.

{{</ hint >}}

### Installing CoreDNS as Cluster DNS

To provide [DNS resolving for pods and service names](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) it is recommended to run CoreDNS on your cluster. It can be installed from upstream Helm chart.

To install it, run the following command:

```sh
helm upgrade --install -n kube-system coredns stable/coredns --set rbac.pspEnable=true --set service.ClusterIP=$DNS_SERVICE_IP
```

### Installing kubelet-rubber-stamp

As part of kubelet TLS bootstrapping process, kubelet requests serving certificate from Kubernetes API, to be able to use it for serving logs and metrics securely to `kube-apiserver`.

At the time of writing, `kube-controller-manager` does not approve those certificates and 3rd party controller needs to be used to automate this process. This is what [kubelet-rubber-stamp](https://github.com/kontena/kubelet-rubber-stamp) does.

It can be installed by running the following command:

```sh
helm upgrade --install -n kube-system kubelet-rubber-stamp flexkube/kubelet-rubber-stamp
```

## Verifying cluster functionality

Now your cluster is ready to use. Go ahead and try deploying some application on it. Please keep following things in mind, while using the cluster:

- Service of type `LoadBalancer` won't get the IP address, as there is no controller, which could assign it.
- The cluster has [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) enabled by default. Make sure your deployment ships the PSP.
- There is no storage provider on the cluster, so pods requesting PVCs will be stuck in pending state.

## Cleaning up

To clean up the host, first, uninstall all helm releases, so kubelet removes all the pods. This can be done using the following command:

```sh
helm uninstall -n kube-system calico coredns kube-proxy kubelet-rubber-stamp tls-bootstrapping
```

Then, rename or remove `config.yaml` file, so CLI will be able to clean up the resources. For example, execute:

```sh
mv config.yaml config.yaml.old
```

Now you can remove all containers managed by `flexkube` using following commands:

```sh
flexkube kubelet-pool default
flexkube controlplane
flexkube etcd
```

Finally, following directories can be removed as well:

```sh
sudo rm -rf /etc/kubernetes/ /var/lib/etcd/ /var/lib/kubelet/ /var/lib/calico/
```

## What's next

This guide explains, how to create a cluster using `flexkube` CLI, which explains every step and provides insights, but might be time consuming and error-prone. For fully automated installation, see "[Creating single-node Kubernetes cluster on local machine using Terraform]({{< relref "/documentation/guides/kubernetes/creating-single-node-cluster-on-local-machine-using-terraform" >}})".

If you want to deploy the cluster to remote machine(s), which also supports HA controlplane, see "[Creating multi-node cluster using Terraform]({{< relref "/documentation/guides/kubernetes/creating-multi-node-cluster-using-terraform" >}})".
