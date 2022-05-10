---
layout: post
title:  "Kube-Vip Load Balancer for On-Premise Kubernetes"
date:   2022-05-10 10:35:00 +0200
categories: kubernetes networking
---

`kube-vip` is an open-source project that aims to simplify providing load balancing services for Kubernetes clusters.

In cloud-based Kubernetes clusters, services are usually exposed by using load balancers provided by cloud vendors. However, cloud-based load balancers are unavailable in bare-metal environments. `kube-vip` allows users to create LoadBalancer Services in bare-metal, edge, and virtualization environments for external access, and provides the same user experience as cloud-based load balancers.

## Installation 

Following documentation, especially [`Installation` section](https://kube-vip.chipzoller.dev/docs/installation/),
there are 2 ways to install it into Kubernetes cluster:

1. [`Static Pods`](https://kube-vip.chipzoller.dev/docs/installation/static/) which is primarily required for `kubeadm` as this is due to the sequence of actions performed by `kubeadm`.

2. [`DaemonSet`](https://kube-vip.chipzoller.dev/docs/installation/daemonset/) when we can apply `kube-vip` to the cluster once the first node has been brought up.

Unlike running `kube-vip` as a static Pod there are a few more things that may need configuring when running `kube-vip` as a `DaemonSet`.

The functionality of `kube-vip` depends on the flags used to create the manifest.

By passing in `--controlplane` we instruct `kube-vip` to provide and advertise a virtual IP (which must be specified via flag `--address`) to be used by the control plane.

By passing in `--services` we tell `kube-vip` to provide load balancing for Kubernetes Service resources created inside the cluster.  It will watch Services of type `LoadBalancer` and once their `spec.LoadBalancerIP` is updated (typically by a cloud controller, including (optionally) the one provided by `kube-vip` in on-prem scenarios) it will advertise this address using BGP/ARP.

By passing in `--inCluster` we instruct `kube-vip` to use `ServiceAccount` called `kube-vip` (`serviceAccountName: kube-vip`). It is required for `kube-vip` as `DaemonSet`.

By passing in `--leaderElection` we instruct `kube-vip` to enable [Kubernetes leader election](https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/) used by ARP, as only the leader can broadcast [ARP](https://wiki.wireshark.org/Gratuitous_ARP)

### [Install the kube-vip Cloud Provider](https://kube-vip.chipzoller.dev/docs/usage/cloud-provider/#install-the-kube-vip-cloud-provider)

The `kube-vip` cloud provider can be used to populate an IP address for Services of type `LoadBalancer` similar to what public cloud providers allow through a Kubernetes CCM.

Install it  using either latest image from Docker Hub:

```
kubectl apply -f https://kube-vip.io/manifests/controller.yaml
```

or latest image from GitHub Packages (ghcr.io):

```
kubectl apply -f https://raw.githubusercontent.com/kube-vip/kube-vip-cloud-provider/main/manifest/kube-vip-cloud-controller.yaml
```

### Create the RBAC settings

Since `kube-vip` as a `DaemonSet` runs as a regular resource, it still needs the correct access to be able to watch Kubernetes Services and other objects. In order to do this, RBAC resources must be created which include a `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` and can be applied this with the command:

```
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
```

###  [Create a global CIDR or IP Range](https://kube-vip.chipzoller.dev/docs/usage/cloud-provider/#create-a-global-cidr-or-ip-range)

```
apiVersion: v1
data:
  range-global: 192.168.0.33-192.168.0.62
kind: ConfigMap
metadata:
  name: kubevip
  namespace: kube-system
```

### [Enable strict ARP in kube-proxy](https://kube-vip.io/kubernetes/arp/)

Use next command to apply changes into `kube-proxy`:

```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### [Generating a Manifest](https://kube-vip.chipzoller.dev/docs/installation/static/#generating-a-manifest)

It is possible to use the `kube-vip` container itself to generate `DaemonSet` manifest. We do this by running the `kube-vip` image as a container and passing in the various [flags](https://kube-vip.chipzoller.dev/docs/installation/flags/) for the capabilities we want to enable.

#### Set configuration details

We use environment variables to predefine the values of the inputs to supply to `kube-vip`.

Set the `VIP` address to be used for the control plane:

```
export VIP=192.168.0.40
```

Get the latest version of the `kube-vip` release by parsing the GitHub API. This step requires that `jq` and `curl` are installed.

```
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
```

To set manually instead, find the desired [release tag](https://github.com/kube-vip/kube-vip/releases):

```
export KVVERSION=v0.4.4
```

#### Creating the manifest

With the input values now set, we can pull and run the `kube-vip` image supplying it the desired flags and values.

Depending on the container runtime, use one of the two commands to create a `kube-vip` command which runs the `kube-vip` image as a container.

For containerd, run the below command (see [Creating the manifest](https://kube-vip.chipzoller.dev/docs/installation/static/#creating-the-manifest) of original documentation):

```
alias kube-vip="ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

For Docker, run the below command:

```
alias kube-vip="docker run --rm --network host ghcr.io/kube-vip/kube-vip:$KVVERSION"
```

#### ARP

With the inputs and alias command set, we can run the `kube-vip` container to generate a manifest. This is assumed to run on the control plane node.

`kube-vip` can optionally be configured to broadcast a Gratuitous ARP that will typically immediately notify all local hosts that the VIP-to-MAC address mapping has changed.

This configuration will create a manifest that starts `kube-vip` providing Kubernetes Service management using the `leaderElection` method and ARP. When this instance is elected as the leader, it will bind the `vip` to default network interface. This is the same behavior for Services of type `LoadBalancer`.

When creating the `kube-vip` installation manifest as a `DaemonSet`, the manifest subcommand takes the value `daemonset`. The flag `--inCluster` is also needed to configure the `DaemonSet` to use a `ServiceAccount`.

```
kube-vip manifest daemonset \
    --inCluster \
    --services \
    --arp \
    --leaderElection
```

#### Example ARP Manifest

```
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app.kubernetes.io/name: kube-vip-ds
    app.kubernetes.io/version: v0.4.4
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-vip-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: kube-vip-ds
        app.kubernetes.io/version: v0.4.4
    spec:
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: svc_enable
          value: "true"
        - name: vip_leaderelection
          value: "true"
        image: ghcr.io/kube-vip/kube-vip:v0.4.4
        imagePullPolicy: Always
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
      hostNetwork: true
      serviceAccountName: kube-vip
  updateStrategy: {}
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
```

#### BGP

This configuration will create a manifest that starts `kube-vip` providing Kubernetes Service management. Unlike ARP, all nodes in the BGP configuration will advertise virtual IP addresses.

> Note: we bind the address to `lo` as we don't want multiple devices that have the same address on public interfaces. We must specify at least one peer in a comma-separated list in the format of `address:AS:password:multihop`. The `routerID` needs to be specified as well.

```
export INTERFACE=lo
```

```
kube-vip manifest daemonset \
    --interface $INTERFACE \
    --inCluster \
    --services \
    --bgp \
    --bgpRouterID 192.168.0.1 \
    --bgppeers 192.168.0.10:65000::false,192.168.0.11:65000::false,192.168.0.12:65000::false
```

#### Example BGP Manifest

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app.kubernetes.io/name: kube-vip-ds
    app.kubernetes.io/version: v0.4.4
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-vip-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: kube-vip-ds
        app.kubernetes.io/version: v0.4.4
    spec:
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "false"
        - name: vip_interface
          value: lo
        - name: svc_enable
          value: "true"
        - name: bgp_enable
          value: "true"
        - name: bgp_routerid
          value: 192.168.0.1
        - name: bgp_as
          value: "65000"
        - name: bgp_peeras
          value: "65000"
        - name: bgp_peers
          value: 192.168.0.10:65000::false,192.168.0.11:65000::false,192.168.0.12:65000::false
        image: ghcr.io/kube-vip/kube-vip:v0.4.4
        imagePullPolicy: Always
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
      hostNetwork: true
      serviceAccountName: kube-vip
  updateStrategy: {}
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
```

### Creating manifest on CRI-O

`crictl` command operates with Pods, therefore it is difficult to use `crictl run`

Instead it is better to create Kubernetes manifests.

#### ARP

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-vip-manifest-ds
spec:
  selector:
    matchLabels:
      app: kube-vip-manifest-arp
  replicas: 1
  template:
    metadata:
      labels:
        app: kube-vip-manifest-arp
    spec:
      containers:
      - name: kube-vip-manifest
        image: ghcr.io/kube-vip/kube-vip:v0.4.4
        command:
         - '/kube-vip'
         - 'manifest'
         - 'daemonset'
         - '--services'
         - '--arp'
         - '--inCluster'
         - '--leaderElection'
      hostNetwork: true
```

#### BGP

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-vip-manifest-ds
spec:
  selector:
    matchLabels:
      app: kube-vip-manifest-bgp
  replicas: 1
  template:
    metadata:
      labels:
        app: kube-vip-manifest-bgp
    spec:
      containers:
      - name: kube-vip-manifest
        image: ghcr.io/kube-vip/kube-vip:v0.4.4
        command:
         - '/kube-vip'
         - 'manifest'
         - 'daemonset'
         - '--inCluster'
         - '--interface'
         - 'lo'
         - '--services'
         - '--bgp'
         - '--bgpRouterID'
         - '192.168.0.1'
         - '--bgppeers'
         - '192.168.0.6:65000::false,192.168.0.7:65000::false,192.168.0.9:65000::false'
      hostNetwork: true
```

To apply this manifest, use next command:

```
kubectl apply -f kube-vip-manifest-ds.yaml 
```

To get just generated manifest, use next command:

```
kubectl logs deploy/kube-vip-manifest-ds > kube-vip-ds-bgp.yaml
```

## Testing

Deploy the [Hello EKS Anywhere](https://anywhere.eks.amazonaws.com/docs/tasks/workload/test-app/) test application.

```
kubectl apply -f "https://anywhere.eks.amazonaws.com/manifests/hello-eks-a.yaml"
```

### Create load balancer service:

```
---
apiVersion: v1
kind: Service
metadata:
  name: hello-eks-a
spec:
  type: LoadBalancer
  selector:
    app: hello-eks-a
  ports:
    - port: 80
  loadBalancerIP: 192.168.0.50
```

### Test it 

```
curl http://192.168.0.50
```