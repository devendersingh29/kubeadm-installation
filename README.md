# Kubernetes Cluster Setup Guide
> 1 Master Node + 2 Worker Nodes on Ubuntu (AWS EC2)

---

## Prerequisites
- 3 Ubuntu EC2 instances (1 Master + 2 Workers)
- All nodes me ports open hone chahiye (Security Group):
  - `6443` — Kubernetes API Server
  - `10250` — Kubelet
  - `10259`, `10257` — Controller Manager & Scheduler
  - `2379`, `2380` — etcd

---

## Step 1: Script Run Karo (Teeno Nodes Pe)

Ye script Docker + Kubernetes install karegi.

```bash
#!/bin/bash
set -e

# Update packages
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Docker repository add karo
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker install
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker

# Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Kubernetes install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
sudo apt-mark hold kubelet kubeadm kubectl

# Swap disable
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kernel modules
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
echo -e "net.bridge.bridge-nf-call-ip6tables = 1\nnet.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/kubernetes.conf
sudo sysctl --system

# AppArmor stop
sudo systemctl stop apparmor

# Containerd configure (IMPORTANT - SystemdCgroup = true)
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Containerd restart
sudo systemctl restart containerd
```

> ⚠️ **Important:** `SystemdCgroup = true` zaroori hai — bina iske API server start nahi hoga!

---

## Step 2: Master Node Pe — Cluster Init Karo

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
sudo kubeadm init
```
```bash
sudo kubeadm init
```

---

## Step 3: kubectl Setup (Master Pe)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
unset KUBECONFIG
```

---

## Step 4: Join Command Save Karo (Master Pe)

`kubeadm init` ke output mein ye milega — **copy karke save karo:**

```bash
kubeadm join <master-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Agar miss ho jaye, regenerate karo:

```bash
kubeadm token create --print-join-command
```

---

## Step 5: Worker Nodes Pe Reset Karo (Agar Pehle Init Chala Ho)

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/ /var/lib/etcd/ /var/lib/kubelet/
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -X
```

---

## Step 6: Worker Nodes Ko Join Karo (Dono Workers Pe)

```bash
sudo kubeadm join <master-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Step 7: CNI Plugin Lagao — Calico (Master Pe)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

---

## Step 8: Verify Karo (Master Pe)

```bash
# Nodes check karo
kubectl get nodes

# Pods check karo
kubectl get pods -n kube-system
```

Expected output:
```
NAME               STATUS   ROLES           AGE   VERSION
ip-172-31-xx-xxx   Ready    control-plane   10m   v1.29.15
ip-172-31-xx-xxx   Ready    <none>          5m    v1.29.15
ip-172-31-xx-xxx   Ready    <none>          5m    v1.29.15
```

---

## Common Errors & Fixes

| Error | Fix |
|---|---|
| `connection refused 6443` | `SystemdCgroup = true` set nahi hai containerd me |
| `permission denied admin.conf` | `unset KUBECONFIG` run karo |
| `port already in use` | `sudo kubeadm reset -f` phir dobara init karo |
| `nodes NotReady` | CNI plugin (Calico) nahi laga |
| `FileAvailable errors` | Reset karo: `sudo kubeadm reset -f` |

---

## Node Roles Summary

| Node | Command |
|---|---|
| Master (1 node only) | `sudo kubeadm init` ✅ |
| Worker (all others) | `sudo kubeadm join ...` ✅ |
| Worker pe init | ❌ Kabhi nahi |

---

## Pod Network CIDR Reference

| CNI Plugin | CIDR |
|---|---|
| Calico | `192.168.0.0/16` |
| Flannel | `10.244.0.0/16` |
| Weave | `10.32.0.0/12` |
