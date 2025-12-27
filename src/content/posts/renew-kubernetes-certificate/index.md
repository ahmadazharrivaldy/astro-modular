---
title: Renew Kubernetes Certificates
date: 2025-12-22
description: ""
tags:
  - kubernetes
  - devops
image: attachments/kubernetes-banner.png
imageAlt: ""
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword: ""
draft: false
---
After creating a Kubernetes cluster, you must track the certificate expiration dates. If these certificates expire, communication between Kubernetes components and APIs will fail, leading to cluster instability. Maintaining these certificates is essential to ensure continuous uptime.
## Backup Kubernetes Certificates

Check expiration date of Kubernetes certificates

```bash
## for > v1.20
kubeadm certs check-expiration

## for < v1.20
kubeadm alpha certs check-expiration
```

Run a health check before renewing your Kubernetes certificates

```bash
kubectl get --raw='/readyz?verbose'
```

Backup Kubernetes certificates

```bash
mkdir -p backup-kube-certs/temp
sudo cp -r /etc/kubernetes/ ~/backup-kube-certs
```

## Renew Kubernetes Certificates

Renew all Kubernetes certificates

```bash
## for > v1.20
kubeadm certs renew all

## for < v1.20
kubeadm alpha certs renew all
```

Restart kube-system pods to apply new certificates

```bash
sudo mv /etc/kubernetes/manifests/ ~/backup-kube-certs/temp/
sleep 120
sudo mv ~/backup-kube-certs/temp/ /etc/kubernetes/manifests/
```

Verify that the age of the pods has been changed, as this indicates that the pods have been restarted.

```bash
kubectl -n kube-system get pods -o wide --field-selector=spec.nodeName=CHANGE_WITH_YOUR_MASTER_NODE_NAME | egrep "kube-apiserver|kube-controller-manager|kube-scheduler|etcd"
```

```bash
## for containerd
crictl ps | egrep "kube-apiserver|kube-controller-manager|kube-scheduler|etcd"

## for docker
docker ps | egrep "kube-apiserver|kube-controller-manager|kube-scheduler|etcd"
```