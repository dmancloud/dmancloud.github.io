---
layout: post
title: "Install Rancher Manager With Lets Encrypt"
date: 2022-11-10 22:00:00 -0500
categories: [rancher]
tags: [kubernetes,k8s,cert-manager,rancher,helm,ci/cd]
#image:
#  path: /assets/img/headers/mirror-image-chess.jpg
---

Rancher is a popular open-source platform for managing and deploying containerized applications on Kubernetes. By using cert-manager and Let's Encrypt, we will ensure that all communication with the Rancher server is secure and encrypted.

### Watch the tutorial
{% include embed/youtube.html id='IdvhywzV1oo' %}
▶️ [Watch On YouTube](https://youtu.be/IdvhywzV1oo)

## Prerequsites
- Domain Name
- Ability to make DNS Changes
- Ubuntu Virtual Machine
- Port 80 & 443 must be accessible for Let's Encrypt to verify and issue certificates

## Configure DNS
Pick a subdomain and create a DNS entry pointing to the IP Address that will be assigned to the Rancher Server

``` sh
curl -4 icanhazip.com
```

Head over to your DNS Provider and create an ````A```` record

``` sh
dig +short replace_with_subdomain
```

## Update Package Repository and Upgrade Packages

``` sh
sudo apt-get update && sudo apt upgrade -y
```

## Install required packages

``` sh
sudo apt-get -y install gnupg2 ca-certificates curl apt-transport-https iptables
```

### Install Helm v3

Additional Information - https://helm.sh/docs/intro/install/

``` sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Install kubectl

Additional Information - https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
``` sh
sudo apt update
sudo apt install ca-certificates curl apt-transport-https -y
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install kubectl -y
```

## Install RKE2 Cluster

Additional Information - https://docs.rke2.io/

``` sh
sudo bash
curl -sfL https://get.rke2.io | sh -
exit
```

``` sh
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
sudo journalctl -u rke2-server -f
mkdir ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 400 ~/.kube/config
kubectl get pods -A
```

### Install cert-manager

Additional Information - https://cert-manager.io/docs/installation/

``` sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager  --namespace cert-manager --create-namespace --set installCRDs=true
```

### Install Rancher

Additional Information - https://docs.ranchermanager.rancher.io/

``` sh
kubectl create ns cattle-system
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=HOSTNAME --set bootstrapPassword=PASSWORD --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=EMAIL_ADDRESS --set letsEncrypt.ingress.class=nginx
kubectl -n cattle-system rollout status deploy/rancher
watch kubectl get pods -A
```

### Access Rancher User Interface

``` sh
https://RANCHER_URL
```