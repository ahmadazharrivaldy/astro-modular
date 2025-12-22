---
title: Create Persistent Volume with Ceph RBD
date: 2025-12-22
description: ""
tags:
  - kubernetes
  - ceph
  - devops
image: ""
imageAlt: ""
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword: ""
draft: false
---
In my environment I already have Ceph cluster, so i decided to use it for persistent volume backend storage on my Kubernetes cluster. So how i create it? lets go to the tutorial.

## Environment
### Ceph cluster

| Hostname    | IP Address    |
| ----------- | ------------- |
| ceph-node01 | 192.168.10.30 |
| ceph-node02 | 192.168.10.31 |
| ceph-node03 | 192.168.10.32 |
### Kubernetes cluster

| Hostname | IP Address    |
| -------- | ------------- |
| master   | 192.168.10.10 |
| worker01 | 192.168.10.11 |
| worker02 | 192.168.10.12 |

## Deploy Ceph CSI RBD

Get the fsid and Ceph mon IP Address

```bash
ceph mon dump
...
dumped monmap epoch 4
epoch 4
fsid f2e8791a-37ad-11f0-b942-15fd889ba33d
last_changed 2025-05-23T08:20:46.148011+0000
created 2025-05-23T08:14:57.778245+0000
min_mon_release 15 (octopus)
0: [v2:192.168.10.30:3300/0,v1:192.168.10.30:6789/0] mon.ceph-node01
1: [v2:192.168.10.31:3300/0,v1:192.168.10.31:6789/0] mon.ceph-node02
```

Add Ceph CSI RBD repository

```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update
```

Create osd pool for Kubernetes persistent volume

```bash
ceph osd pool create kubernetes 64 64
rbd pool init kubernetes
ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
```

Get user key

```bash
ceph auth get-key client.kubernetes | base64
...
QVFEWUkxVm9EVnRFRXhBQWRwcW1adExpdXFIaEJYQjduYXVRL3c9PQ==
```

Export helm values

```bash
helm inspect values ceph-csi/ceph-csi-rbd > ceph-csi-rbd-values.yaml
```

Edit helm values and change with our Ceph information

```yaml
...
csiConfig:
  - clusterID: "CHANGE_WITH_YOUR_FSID"
    monitors:
      - "192.168.10.10:6789"
      - "192.168.10.11:6789"
...
storageClass:
  create: true
  name: csi-rbd-sc
  clusterID: CHANGE_WITH_YOUR_FSID
  pool: kubernetes
  imageFeatures: "layering"
  provisionerSecret: csi-rbd-secret
  provisionerSecretNamespace: ""
  controllerExpandSecret: csi-rbd-secret
  controllerExpandSecretNamespace: ""
  nodeStageSecret: csi-rbd-secret
  nodeStageSecretNamespace: ""
  fstype: ext4
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  mountOptions:
    - discard
...
secret:
  create: true
  name: csi-rbd-secret
  annotations: {}
  userID: kubernetes
  userKey: CHANGE_WITH_YOUR_USERKEY
...
```

Deploy it

```bash
kubectl create namespace ceph-csi-rbd
helm install --namespace ceph-csi-rbd ceph-csi-rbd ceph-csi/ceph-csi-rbd --values ceph-csi-rbd-values.yaml
```

Try to create persistent volume

```bash
cat > ceph-rbd-sc-pvc.yaml <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-rbd-sc-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: csi-rbd-sc
EOF
```

```bash
kubectl apply -f ceph-rbd-sc-pvc.yaml
```

## Verify

Make sure your persistent volume claim is bound and persistent volume automatically created

```bash
kubectl get pvc
...
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-rbd-sc-pvc   Bound    pvc-c02f6a88-a965-4440-96c6-dc6d8e9340cd   2Gi        RWO            csi-rbd-sc     89s
```

```bash
kubectl get pv
...
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
pvc-c02f6a88-a965-4440-96c6-dc6d8e9340cd   2Gi        RWO            Delete           Bound    default/ceph-rbd-sc-pvc   csi-rbd-sc              6s
```

RBD image has been created in kubernetes pool

```bash
rbd -p kubernetes ls
...
csi-vol-a0c6862a-7188-4d1d-bc1e-b12ceb79929c
```

```bash
rbd info kubernetes/csi-vol-a0c6862a-7188-4d1d-bc1e-b12ceb79929c
...
rbd image 'csi-vol-a0c6862a-7188-4d1d-bc1e-b12ceb79929c':
        size 2 GiB in 512 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 34501f714dc9b
        block_name_prefix: rbd_data.34501f714dc9b
        format: 2
        features: layering
        op_features:
        flags:
        create_timestamp: Tue Jun 24 11:29:48 2025
        access_timestamp: Tue Jun 24 11:29:48 2025
        modify_timestamp: Tue Jun 24 11:29:48 2025
```