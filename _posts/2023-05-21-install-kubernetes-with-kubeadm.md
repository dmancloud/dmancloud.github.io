---
layout: post
title: "How To Setup Kubernetes Cluster Using Kubeadm"
date: 2023-05-21 09:00:00 -0500
categories: [kubernetes]
tags: [kubernetes,k8s,kubeadm]
#image:
#  path: /assets/img/headers/mirror-image-chess.jpg
---
[Kubeadm](https://github.com/kubernetes/kubeadm) is an excellent tool to set up a working Kubernetes cluster in less time. It does all the heavy lifting in terms of setting up all Kubernetes cluster components. Also, It follows all the configuration best practices for a Kubernetes cluster.

Kubeadm Setup Prerequisites
---------------------------

Following are the prerequisites for Kubeadm Kubernetes cluster setup.

1.  Minimum three Debian 12 /Ubuntu 22 nodes [One master and two worker nodes]. You can have more worker nodes as per your requirement.
2.  The master node should have a minimum of 2 vCPU and 2GB RAM.
3.  For the worker nodes, a minimum of 1vCPU and 2 GB RAM is recommended.
4.  192.168.1.X/24 network range with static IPs for master and worker nodes. We will be using the 10.1.x.x series as the pod network range that will be used by the Calico network plugin. Make sure the Node IP range and pod IP range don't overlap.

## Enable iptables Bridged Traffic on all the Nodes
Execute the following commands on all the nodes for IPtables to see bridged traffic.
``` sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```sh
sudo modprobe overlay
sudo modprobe br_netfilter
```

sysctl params required by setup, params persist across reboots
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Apply sysctl params without reboot
```sh
sudo sysctl --system
```
## Disable swap on all the Nodes
For kubeadm to work properly, you need to disable swap on all the nodes using the following command.
```sh
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```
The `fstab` entry will make sure the swap is off on system reboots.

You can also, control swap errors using the kubeadm parameter `--ignore-preflight-errors` Swap we will look at it in the latter part.

## Install CRI-O Runtime On All The Nodes

The basic requirement for a Kubernetes cluster is a container runtime. You can have any one of the following container runtimes.

1.  CRI-O
2.  containerd
3.  Docker Engine (using cri-dockerd)

We will be using CRI-O instead of Docker for this setup as [Kubernetes deprecated Docker engine](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)

As a first step, we need to install `cri-o` on all the nodes. Execute the following commands on all the nodes.

Create the `.conf `file to load the modules at bootup
```sh
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF
```
Set up required sysctl params, these persist across reboots.
```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
Execute the following commands to enable overlayFS & VxLan pod communication.
```sh
sudo modprobe overlay
sudo modprobe br_netfilter
```
Set up required sysctl params, these persist across reboots.
```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
Reload the parameters.
```sh
sudo sysctl --system
```
Enable cri-o repositories for version 1.28
```sh
export OS="Debian_12"
export VERSION="1.28"
```
```sh
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
```
```sh
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF
```
Add the gpg keys.
```sh
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
```
```sh
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
```
Update and install crio and crio-tools.
```sh
sudo apt-get update
sudo apt-get install cri-o cri-o-runc cri-tools -y
```
Reload the systemd configurations and enable cri-o.
```sh
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```
## Install Kubeadm & Kubelet & Kubectl on all Nodes
Install the required dependencies.
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```
Add the GPG key and apt repository.
```sh
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update apt and install the latest version of kubelet, kubeadm, and kubectl.
```sh
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```
Add hold to the packages to prevent upgrades.
```sh
sudo apt-mark hold kubelet kubeadm kubectl
```
Now we have all the required utilities and tools for configuring Kubernetes components using kubeadm.

Add the node IP to `KUBELET_EXTRA_ARGS`. Make sure you select the correct network interface for your virtual machine. In this example it is `ens192`

```sh
sudo apt-get install -y jq
local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "ens192" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```
## Initialize Kubeadm On Master Node To Setup Control Plane
Set the following environment variables. Replace `192.168.1.25` with the IP of your master node.
```sh
export IPADDR="192.168.1.25"
export NODENAME=$(hostname -s)
export POD_CIDR="10.1.0.0/16"
```
Now, initialize the master node control plane configurations using the kubeadm command.

```sh
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
```

On a successful kubeadm initialization, you should get an output with kubeconfig file location and the join command with the token as shown below. Copy that and save it to the file. we will need it for joining the worker node to the master.

### Test cluster access
Use the following commands from the output to create the `kubeconfig` in master so that you can use `kubectl` to interact with cluster API.

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Now, verify the kubeconfig by executing the following kubectl command to list all the pods in the `kube-system` namespace.
```sh
kubectl get po -n kube-system
```
You should see the following output. You will see the two Coredns pods in a pending state. It is the expected behavior. Once we install the network plugin, it will be in a running state

You can get the cluster info using the following command.
```sh
kubectl cluster-info
```
## Install Calico Network Plugin for Pod Networking
Kubeadm does not configure a network plugin. You need to install a network plugin of your choice.

In this tutorial we are using the Calico network plugin.

```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
```

## Join Worker Nodes To Kubernetes Master Node
Execute the following command on the master node to create the token with the join command.
```sh
kubeadm token create --print-join-command
```

Run this command on the nodes, this will perform the TLS bootstrapping for each of the nodes.

Now execute the kubectl command from the master node to check if the node is added to the master.
```sh
kubectl get nodes
```

Example output,
```ss
root@master-node:/home/vagrant# kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
master-node     Ready    control-plane   14m     v1.27.2
worker-node01   Ready    <none>          2m13s   v1.27.2
worker-node02   Ready    <none>          2m5s    v1.27.2
```

## Setup Kubernetes Metrics Server (Optional)
Kubeadm does not install the metrics server during its initialization. Let's install it.

We will need to modify the official metrics server and add the `--kubelet-insecure-tls=true` flag to the container to make it work 

```sh
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Next we need to edit the file `components.yaml` find the image section and add the `insecure flag`

```sh 
vim components.yaml
```
Find and add
```yml
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --kubelet-insecure-tls=true #add this line
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: registry.k8s.io/metrics-server/metrics-server:v0.6.3
```

```sh
kubectl apply -f components.yaml
```

Now after a few minutes you can check to see if metrics-server is reporting correctly

```sh
kubectl top node
```