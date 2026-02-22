# Kubernetes Worker Node Setup Guide

> **Cluster Type:** Bare-metal kubeadm Kubernetes (upstream)
> **OS:** Ubuntu Server 22.04 LTS
> **Kubernetes Version:** v1.31
> **Container Runtime:** containerd
> **CNI Plugin:** Flannel

---

## What ? Why? Were ?

I am installing that for each K8S node in proxmox cluster.
Each Proxmox node has 1 VM with Ubuntu Server , and that is a node for K8S. 

---

## Prerequisites

- A running kubeadm control plane node
- A fresh Ubuntu Server 22.04 LTS VM with network access to the control plane
- `sudo` privileges on the new node

---

## Step 1 — Disable Swap

Kubernetes requires swap to be disabled for the kubelet to function correctly.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## Step 2 — Load Required Kernel Modules

Enable the `overlay` and `br_netfilter` modules needed for container networking.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## Step 3 — Configure Sysctl Parameters

Enable bridge networking and IP forwarding for pod-to-pod communication across nodes.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## Step 4 — Install and Configure containerd

Install the container runtime and configure it to use the `systemd` cgroup driver, which is required for compatibility with the kubelet.

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

## Step 5 — Install kubeadm, kubelet, and kubectl

Add the Kubernetes APT repository and install the required packages.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl conntrack
sudo apt-mark hold kubelet kubeadm kubectl
```

> **Note:** `apt-mark hold` prevents these packages from being automatically upgraded, which could break cluster compatibility.

---

## Step 6 — Join the Cluster

### Generate a join token (run on the control plane)

```bash
kubeadm token create --print-join-command
```

### Join the worker node (run on the new node)

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## Step 7 — Label the Worker Node (run on the control plane)

```bash
kubectl label node <NEW_NODE_NAME> node-role.kubernetes.io/worker=
```

---

## Step 8 — Verify

```bash
kubectl get nodes
```

All nodes should show a `Ready` status. If the new node shows `NotReady`, wait a minute for Flannel to finish setting up networking.

---

## Component Summary

| Component      | Purpose                                                        |
|----------------|----------------------------------------------------------------|
| **containerd** | Container runtime — runs and manages containers on the node    |
| **kubelet**    | Node agent — manages pods and communicates with the API server |
| **kubeadm**    | Cluster lifecycle tool — init, join, reset, upgrade            |
| **kubectl**    | CLI client for interacting with the Kubernetes API             |
| **Flannel**    | CNI plugin — provides pod-to-pod networking across nodes       |
| **conntrack**  | Kernel connection tracking — required by kube-proxy            |

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| Node shows `NotReady` | Wait 1–2 minutes for Flannel to initialize, then check `kubectl describe node <name>` |
| `ca.crt already exists` error on join | Run `sudo kubeadm reset -f` before joining |
| Node name already exists in cluster | Delete the old entry with `kubectl delete node <name>`, then reset and rejoin |
| Missing `conntrack` | Install with `sudo apt-get install -y conntrack` |

---
