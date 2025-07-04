# RKE2 Kubernetes Cluster Setup

## Project Description

This repository contains a step-by-step guide and reference scripts for setting up an RKE2-based Kubernetes cluster on KVM virtual machines using `virsh`. It walks through preparing VMs, setting up a control-plane node, and joining worker nodes. Ideal for private cloud or lab environments.

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [VM Creation with Virsh](#vm-creation-with-virsh)
- [Networking Setup](#networking-setup)
- [Installing RKE2 - Master Node](#installing-rke2---master-node)
- [Installing RKE2 - Worker Nodes](#installing-rke2---worker-nodes)
- [Verifying the Cluster](#verifying-the-cluster)
- [Post-Installation Tips](#post-installation-tips)

---

## Overview

This guide helps you:

- Prepare Ubuntu-based VMs using `virsh`
- Install and configure RKE2 (Rancher Kubernetes Engine 2)
- Form a multi-node Kubernetes cluster (1 master, 2+ workers)

---

## Requirements

- Host system with KVM/QEMU and `libvirt`
- Ubuntu 22.04 LTS or later as guest OS for all nodes
- Internet access to pull RKE2 scripts and container images
- sudo privileges on guest VMs

---

## VM Creation with Virsh

1. Clone base Ubuntu image:

   ```bash
   virsh clone --original <base-image> --name RKE2_Master-1
   virsh clone --original <base-image> --name RKE2_Worker-1
   virsh clone --original <base-image> --name RKE2_Worker-2
   ```

2. Start the domain:

   ```bash
   virsh start RKE2_Master-1
   ```

3. Resize disk if needed:

   ```bash
   virsh blockresize RKE2_Master-1 vda +20G
   ```

4. Attach NIC:

   ```bash
   virsh attach-interface --domain RKE2_Master-1 --type network --source default --model virtio --config --live
   ```

5. Console access:

   ```bash
   virsh console RKE2_Master-1
   ```

6. Set up static IP:

   ```bash
   sudo nmcli connection modify 'Wired connection 1' \
     ipv4.method manual \
     ipv4.addresses <your-ip>> \
     ipv4.gateway <> \
     ipv4.dns 8.8.8.8 \
     ipv6.method disabled

   sudo nmcli connection up 'Wired connection 1'
   ping 8.8.8.8
   ```

7. Set hostname:

   ```bash
   hostnamectl set-hostname rke2-master-1
   ```

8. Open necessary ports using UFW:

   ```bash
   sudo ufw allow 6443/tcp
   sudo ufw allow 9345/tcp
   sudo ufw allow 10250/tcp
   sudo ufw allow 8472/udp
   sudo ufw enable
   ```

![Alt text](Images/ufw-status.png?raw=true "short-nodes")

9. Disable swap:

   ```bash
   sudo swapoff -a
   sed -i '/ swap / s/^/#/' /etc/fstab
   ```
![Alt text](Images/ram-free.png?raw=true "short-nodes")
---

## Installing RKE2 - Master Node

```bash
curl -sfL https://get.rke2.io | sudo sh -

sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
token: <your-secret-token>
tls-san:
  - 10.150.35.181
node-name: rke2-master-1
EOF

sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
sudo journalctl -u rke2-server -f

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc

export PATH=$PATH:/var/lib/rancher/rke2/bin
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
source ~/.bashrc

kubectl get nodes
```

---

## Installing RKE2 - Worker Nodes

Repeat for each worker (with unique `node-name`):

```bash
curl -sfL https://get.rke2.io | sudo sh -
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
server: https://<ip_of_masternode>>:9345
token: <your-secret-token>
node-name: rke2-worker-1
EOF

sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
sudo journalctl -u rke2-agent -f
```

---

## Verifying the Cluster

```bash
kubectl get nodes
```
![Alt text](Images/kbuectl-get-nodes.png?raw=true "pods-in-masternode")

```bash
kubectl get pods -A
```
![Alt text](Images/Pods_Master.png?raw=true "pods-in-masternode")


You should see your control-plane and worker nodes in a `Ready` state.

---
## Bonus Tips

Label Worker Nodes

To assign the role worker to your worker nodes:

```bash
kubectl label node rke2-worker-1 node-role.kubernetes.io/worker=worker
kubectl label node rke2-worker-2 node-role.kubernetes.io/worker=worker
```
![Alt text](Images/Labeling-nodes.png?raw=true "labeling-nodes")
---

## Post-Installation Tips

- Consider setting up a CNI plugin, Helm, and Ingress resources
- Backup `/etc/rancher/rke2` and `/etc/rancher/rke2/rke2.yaml`
- Monitor cluster health using Prometheus, Grafana

---

## License

MIT

## Author

Prashant S B — Feel free to open issues or contribute!

