---
layout: post
title: "ArgoCD Installation"
date: 2022-11-16 21:00:00 -0500
categories: [argocd]
tags: [kubernetes,k8s,cert-manager,argocd,helm,ci/cd]
#image:
#  path: /assets/img/headers/mirror-image-chess.jpg
---

Argo CD is a declarative continuous delivery tool for Kubernetes applications. It uses the GitOps style to create and manage Kubernetes clusters. When any changes are made to the application configuration in Git, Argo CD will compare it with the configurations of the running application and notify users to bring the desired and live state into sync.

### Watch the tutorial
{% include embed/youtube.html id='fBd_tz6BALU' %}
▶️ [Watch On YouTube](https://youtu.be/fBd_tz6BALU)

## Prerequsites 
- Virtual Machine running Ubuntu 22.04 or newer

## Update Package Repository and Upgrade Packages
``` sh
sudo apt update
sudo apt upgrade
```

## Create Kubernetes Cluster
``` sh
sudo bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - --disable traefik
exit 
mkdir .kube
sudo cp /etc/rancher/k3s/k3s.yaml ./config
sudo chown dmistry:dmistry config
chmod 400 config
export KUBECONFIG=~/.kube/config
```

## Install ArgoCD
``` sh 
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
### Change Service to NodePort
Edit the service can change the service type from `ClusterIP` to `NodePort`
``` shell 
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}' 
```
### Fetch Password
``` shell 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Optional (Enable TLS w/Ingress)
If you want to enable access from the internet or private network you can follow the instructions below to install and configure an ingress-controller with lets-encrypt.

``` sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true
```

Create Cluster Issuser for Lets Encrypt `vi letsencrypt-product.yaml` and paste the below contents adjust your email address

``` yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: dinesh@dman.cloud
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

``` sh
kubectl apply -f letsencrypt-product.yaml
```

Deploy nginx-ingress controller
``` sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```

Create ingress for ArgoCD `vi ingress.yaml` and paste the below contents adjust the domain name
``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # If you encounter a redirect loop or are getting a 307 response code
    # then you need to force the nginx ingress to connect to the backend using HTTPS.
    #
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.dev.dman.cloud
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.dev.dman.cloud
    secretName: argocd-secret # do not change, this is provided by Argo CD
```

``` sh
kubectl apply -f ingress.yaml
```

## Install ArgoCD command line tool

Download With Curl
``` sh
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

## Login to ArgoCD from the CLI and change the password
``` sh
argocd login argocd.dev.dman.cloud
```

``` sh
argocd account update-password
```

## Deploy Demo Application

You can use the below repository to deploy a demo nginx application
``` sh
https://github.com/dmancloud/argocd-tutorial
```

## Scale Replicaset 
``` sh 
kubectl scale --replicas=3 deployment nginx -n default
```

## Clean Up
``` sh 
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```
