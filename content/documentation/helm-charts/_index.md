# Helm Charts

Resources provided by Flexkube only allow to run minimal Kubernetes cluster, without many essential services like [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/), [CoreDNS](https://coredns.io/) or [Network Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/). However, those processes can be easily managed using Kubernetes itself, which allows to manage them as any other Kubernetes workload.

It is also recommended to run Kubernetes control plane components (`kube-apiserver`, `kube-scheduler` etc.) as Kubernetes workloads, as this allows easy integration with metrics collection, centralized logging, auto-scaling etc.

The recommended way of installing remaining components is trough `helm` 3.x, which no longer require Tiller for operating. This allows installing Helm charts directly into the Kubernetes temporary control plane. 

## Upstream charts 

Following charts can be used directly from upstream and it is recommended to install them on every cluster:

- [coredns](https://github.com/helm/charts/tree/master/stable/coredns) - provides Cluster DNS service
- [metrics-server](https://github.com/helm/charts/tree/master/stable/metrics-server) - provides API for Pods and Nodes metrics, which is required by `kubectl top` command and auto-scaling

Those charts can be installed from the `stable` repository e.g. using the following command:

```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com/ && \
helm install -n kube-system coredns stable/coredns
```

## Flexkube charts

For the charts, which are not available in upstream projects, Flexkube maintains it's own charts and provides user a repository, from where the charts can be deployed. Here is the list of charts provided by Flexkube:

- [kubernetes](https://github.com/flexkube/kubernetes-helm-chart) - provides `kube-proxy`, `kube-scheduler`, `kube-controller-manager`, extra roles etc.
- [kube-apiserver](https://github.com/flexkube/kube-apiserver-helm-chart) - provides `kube-apiserver`, separately from other Kubernetes components to be able to enforce Kubernetes [version skew policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- [calico](https://github.com/flexkube/calico-helm-chart) - provides [Calico](https://www.projectcalico.org/) CNI
- [kubelet-rubber-stamp](https://github.com/flexkube/kubelet-rubber-stamp-helm-chart) - provides daemon, which approves Kubelet serving certificates, which is not done by `kube-controller-manager` as for other Kubelet certificates

Those charts can be installed from the `flexkube` repository e.g. using the following command:

```sh
helm repo add flexkube https://flexkube.github.io/charts/ && \
helm install -n kube-system calico flexkube/calico
```