How To Setup Kubernetes Cluster Using Kubeadm
=============================================

[Kubeadm](https://github.com/kubernetes/kubeadm) is an excellent tool to set up a working Kubernetes cluster in less time. It does all the heavy lifting in terms of setting up all Kubernetes cluster components. Also, It follows all the configuration best practices for a Kubernetes cluster.

Kubeadm Setup Prerequisites
---------------------------

Following are the prerequisites for Kubeadm Kubernetes cluster setup.

1.  Minimum two Ubuntu nodes [One master and one worker node]. You can have more worker nodes as per your requirement.
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
