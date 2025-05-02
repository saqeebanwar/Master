#!/bin/bash

set -e

KUBE_VERSION="1.28.2"

echo "[Step 1] Updating system and installing dependencies..."
apt-get update && apt-get install -y apt-transport-https ca-certificates curl gpg lsb-release bash-completion

echo "[Step 2] Disabling swap..."
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

echo "[Step 3] Loading kernel modules..."
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

echo "[Step 4] Applying sysctl params..."
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

echo "[Step 5] Installing containerd..."
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

echo "[Step 6] Adding Kubernetes apt repo..."
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

echo "[Step 7] Installing Kubernetes components..."
apt-get update
apt-get install -y kubelet=${KUBE_VERSION}-1.1 kubeadm=${KUBE_VERSION}-1.1 kubectl=${KUBE_VERSION}-1.1
apt-mark hold kubelet kubeadm kubectl

echo "[Step 8] Enabling kubelet service..."
systemctl enable kubelet
systemctl start kubelet

echo "[✅] Kubernetes ${KUBE_VERSION} with containerd installed!"
echo "You can now run: kubeadm init --pod-network-cidr=192.168.0.0/16"
