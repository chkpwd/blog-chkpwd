---
title: "HA - K3S Cluster using Kube-Vip and Metallb"
date: 2023-06-14T10:40:46-04:00
draft: false
---

## High-Availablility Kubernetes Cluster - Kube-Vip and Metallb

**NOTE:** There are multiple ways to install K3s including [`k3sup`](https://k3sup.dev/) or [running the binary](https://rancher.com/docs/k3s/latest/en/quick-start/) locally. Whichever method you choose, the `--tls-san` flag must be passed with the same IP when generating the kube-vip DaemonSet manifest when installing the first server (control plane) instance. This is so that K3s generates an API server certificate with the kube-vip virtual IP address.

**NOTE:**
K3S come with its own service load balancer named Klipper. You need to disable it in order to run MetalLB. To disable Klipper, run the server with the --disable servicelb option, as described in K3s documentation

Read further on to see.

### Initialize the K3s cluster

Join the first master node to the cluster

```bash
# Token for the k3s cluster
SECRET=SUPER_SECRET_TOKEN

curl -fL https://get.k3s.io | sh -s - server \
--token=${SECRET} \
--tls-san 172.16.16.200 \
--disable traefik \
--disable servicelb \
--write-kubeconfig-mode 644 \
--cluster-init
```

After a successful initialization of the cluster. We need to get the kube-vip service up and running.

### Prepare kube-vip manifests deployment

K3s has an optional manifests directory that will be searched to [auto-deploy](https://rancher.com/docs/k3s/latest/en/advanced/#auto-deploying-manifests) any manifests found within. Create this directory first in order to later place the kube-vip resources inside.

```bash
mkdir -p /var/lib/rancher/k3s/server/manifests/
```

RBAC resources are needed to ensure a ServiceAccount exists with those permissions and bound appropriately.

Get the RBAC manifest and place in the auto-deploy directory:

```sh
curl https://kube-vip.io/manifests/rbac.yaml -o /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
```

### Generate a kube-vip DaemonSet Manifest

We use environment variables to predefine the values of the inputs to supply to **kube-vip**.

Set the `VIP` address to be used for the control plane:

```bash
export VIP=172.16.16.200
```

Set the `INTERFACE` name to the name of the interface on the control plane(s) which will announce the VIP.

```bash
export INTERFACE=ens192
```

Get the latest version of the kube-vip release by parsing the GitHub API. This step requires that `jq` and `curl` are installed.

```bash
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
```

Aliased command to create a kube-vip command which runs the kube-vip image as a container.

For **containerd**, run the below command:

```bash
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

#### ARP Example for DaemonSet

When creating the kube-vip installation manifest as a DaemonSet, the `manifest` subcommand takes the value `daemonset` as opposed to the `pod` value. The flags `--inCluster` and `--taint` are also needed to configure the DaemonSet to use a ServiceAccount and affine the kube-vip Pods to control plane nodes thereby preventing them from running on worker instances.

```bash
kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection
```
Refer to the [docs](https://kube-vip.io/docs/usage/k3s/#step-3-generate-a-kube-vip-daemonset-manifest) for more info:

Either store this generated manifest separately in the /var/lib/rancher/k3s/server/manifests/ directory, or append to the existing RBAC manifest called kube-vip-rbac.yaml. As a general best practice, it is a cleaner approach to place all related resources into a single YAML file.

### Joining additional Control Plane(s)

```bash
curl -fL https://get.k3s.io | sh -s - server \
--token=K1051b8cff96c75ada04d408712d6a3fa86ac12e010a86eea3f1fb113fb3d6cc151::server:IAMBATMAN \
--tls-san 172.16.16.200 \
--disable servicelb \
--disable traefik \
--server https://172.16.16.201:6443
```

Repeat in **odd number** sets to ensure **etcd** quorum.

Once all Control Planes have been configured, check node status.

```bash
╭─hyoga@crypto ~ 
╰─$ k get nodes -o wide
NAME         STATUS   ROLES                       AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION    CONTAINER-RUNTIME
kubes-cp-1   Ready    control-plane,etcd,master   42m   v1.26.4+k3s1   172.16.16.201   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-21-amd64   containerd://1.6.19-k3s1
kubes-cp-2   Ready    control-plane,etcd,master   25m   v1.26.4+k3s1   172.16.16.202   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-21-amd64   containerd://1.6.19-k3s1
kubes-cp-3   Ready    control-plane,etcd,master   25m   v1.26.4+k3s1   172.16.16.203   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-21-amd64   containerd://1.6.19-k3s1
```

### Deploying Metallb - L2 Configuration
**Note:** Limitations of L2 setup explained [here.](https://metallb.universe.tf/concepts/layer2/)

To install MetalLB, apply the manifest, configs found [here.](https://metallb.universe.tf/configuration/#layer-2-configuration)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```

Once all **RBAC** are configured, we need to deploy the IP Pool manifests.

```bash
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: metallb-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - metallb-pool
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.16.16.204-172.16.16.206
```

 Deploy manifests.

```bash
kubectl apply -f ipAddressPool.yml -f ipAddressPoolAdvertisement.yml
or 
kubectl apply -f ipPool_l2Advertisments.yml # if merging both manifests
```

Results

```bash
╭─hyoga@crypto ~/code/boilerplates/kubernetes/nginx-http ‹master●› 
╰─$ k get svc -n apps 
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)           AGE
nginx-http-svc   LoadBalancer   10.43.126.241   172.16.16.204   30080:31680/TCP   46s
```
