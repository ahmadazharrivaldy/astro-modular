---
title: Renew Kubernetes Certificate
date: 2025-12-22
description: ""
tags: []
image: attachments/kubernetes-banner.png
imageAlt: ""
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword: ""
draft: false
---
After you create a Kubernetes cluster, you must maintain expiration date of the Kubernetes certificate, because if the certificate expires, communication between Kubernetes components or APIs will fail.

## Backup Kubernetes Certificate

Check expired date of Kubernetes certificate

```bash
## for > v1.20
kubeadm certs check-expiration

## for < v1.20
kubeadm alpha certs check-expiration
```

Health check before renew certificate

```bash
kubectl get --raw='/readyz?verbose'
```

Backup Kubernetes certificate

```bash
mkdir -p backup_k8s_certs/temporary/
sudo cp -r /etc/kubernetes/ ~/backup_k8s_certs
```

## Renew Kubernetes Certificate

Renew all certificate

```bash
## for > v1.20
kubeadm certs renew all

## for < v1.20
kubeadm alpha certs renew all
```

Restart kube-system pods to apply new certificate

```bash
sudo mv /etc/kubernetes/manifests/ ~/backup_k8s_certs/temporary/
sleep 120
sudo mv ~/backup_k8s_certs/temporary/ /etc/kubernetes/manifests/
```

Make sure age of pods is changed, because this is indication these pods is restarted.

```bash
kubectl -n kube-system get pods -o wide --field-selector=spec.nodeName=CHANGE_WITH_YOUR_MASTER_NODE_NAME
```

```bash
## for containerd
crictl ps | egrep "kube-apiserver|kube-controller-manager|kube-scheduler|etcd"

## for docker
docker ps | egrep "kube-apiserver|kube-controller-manager|kube-scheduler|etcd"
```