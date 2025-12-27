---
title: Build OpenStack with Kolla Ansible
date: 2025-12-27
description: ""
tags: []
image: attachments/cloud.png
imageAlt: ""
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword: ""
draft: false
---
To simplify my OpenStack deployment, I chose Kolla. It provides production-ready Docker containers and Ansible playbooks, making it one of the most user-friendly tools for managing the lifecycle of an OpenStack cluster.

## Environment

| Hostname   | IP Address                                  | Disk                                             |
| ---------- | ------------------------------------------- | ------------------------------------------------ |
| controller | 192.168.20.10 (ens3)<br>192.168.30.X (ens4) | 100GB (vda - root)                               |
| compute01  | 192.168.20.11 (ens3)                        | 50GB (vda - root)<br>50GB (vdb - cinder-volumes) |
| compute02  | 192.168.20.12 (ens3)                        | 50GB (vda - root)<br>50GB (vdb - cinder-volumes) |

## Setup on Compute Host

### Create cinder-volumes vgroup

Update and upgrade the linux package lists

```bash
sudo apt update && sudo apt upgrade -y
```

Create a vgroup

```bash
pvcreate /dev/vdb
vgcreate cinder-volumes /dev/vdb
```

Verify that a vgroup is exist

```bash
vgs
```

## Setup on Controller Host

### Install Dependencies

Update and upgrade the linux package lists

```bash
sudo apt update && sudo apt upgrade -y
```

Install python dependencies

```bash
sudo apt install -y git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev python3-venv
```

### Create an Empty IP Address for External Network

Edit network configuration

```bash
nano /etc/netplan/50-cloud-init.yaml
```

Remove IP address on ens4

```bash
network:
  version: 2
  ethernets:
    ens3:
      addresses:
      - "192.168.20.10/24"
      nameservers:
        addresses:
        - 8.8.8.8
        - 8.8.4.4
      gateway4: 192.168.20.1
      dhcp4: false
      mtu: 1500
    ens4:
#      addresses:
#      - "192.168.30.10/24"
      nameservers:
        addresses:
        - 8.8.8.8
        - 8.8.4.4
      dhcp4: false
      mtu: 1500
```

Apply the configuration

```bash
netplan apply
```

### Create a Virtual Environment

Create a virtual environment and activate

```bash
python3 venv kolla-env
source kolla-env/bin/activate
```

Uprade pip version to latest

```bash
pip install -U pip
```

### Install Kolla Ansible

Install kolla ansible and its dependencies

```bash
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.2
```

Create kolla directory

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy global configuration and password configuration to kolla directory

```bash
cp -r kolla-env/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

Copy inventory file to the current directory, it contains multinode and all-in-one files.

```bash
cp kolla-env/share/kolla-ansible/ansible/inventory/* .
```

Install Ansible galaxy dependencies

```bash
kolla-ansible install-deps
```

### Setup Configuration

Edit ansible configuration

```bash
sudo mkdir -p /etc/ansible
sudo nano /etc/ansible/ansible.cfg
```

Adjust the fork value, don't set it too high or too low. Adjust it according to your network bandwidth.

```bash
[defaults]
host_key_checking=False
pipelining=True
forks=20
```

Edit inventory, in my case i choose multinode.

```bash
nano multinode
```

```yaml
...
[control]
controller

[network]
controller

[compute]
compute01
compute02

[monitoring]
controller

[storage]
compute01
compute02

[deployment]
localhost       ansible_connection=local
...
```

Generate openstack password

```bash
kolla-genpwd
```

Edit global configuration

```bash
nano /etc/kolla/global.yml
```

```yaml
...
workaround_ansible_issue_8743: yes
kolla_base_distro: "ubuntu"
kolla_internal_vip_address: "192.168.20.100"
network_interface: "ens3"
neutron_external_interface: "ens4"
enable_openstack_core: "yes"
enable_haproxy: "yes"
enable_mariadb: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
cinder_cluster_name: "cinder-volumes"
...
```

Ensure access between hosts is accessible without permission restrictions like passwords, so you can create an RSA key first and import it to each host.

```bash
ssh-keygen -t rsa
ssh-copy-id root@controller
ssh-copy-id root@compute01
ssh-copy-id root@compute02
```

Bootstap the hosts

```bash
kolla-ansible bootstrap-servers -i ~/multinode
```

Run pre-checks to ensure there are no missconfiguration

```bash
kolla-ansible prechecks -i ~/multinode
```

Proceed to actual OpenStack deployment, it takes up to 1 hour.

```bash
kolla-ansible deploy -i ~/multinode
```

### Verify

Install Openstack client CLI

```bash
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2025.2
```

Generate `clouds.yaml` file where credentials for the admin user are set

```bash
kolla-ansible post-deploy
```

Verify the cluster to run this command

```bash
cp /etc/kolla/admin-openrc.sh .
source admin-openrc.sh
openstack endpoint list
openstack network agent list
openstack volume service list 
openstack compute service list
```