# Kubernetes (kubeadm) Local Cluster on Fedora 43

This repository documents the **end-to-end setup, maintenance, reset, and troubleshooting**
of a **single-node Kubernetes cluster** installed using **kubeadm** on **Fedora 43**.

This setup is intentionally close to **production Kubernetes** for learning Kubernetes
internals, SRE workflows, and real failure modes.

---

## Cluster Characteristics

- OS: Fedora Linux 43
- Kubernetes: v1.29.x (pinned)
- Install method: kubeadm (official)
- Container runtime: containerd
- Networking: Calico CNI
- Metrics: metrics-server
- Mode: Single-node (control plane + workloads)

---

## Architecture Overview

- Control plane runs as **static pods**
- kubelet manages all core components
- etcd runs locally (static pod)
- Docker is retained **only for image builds**
- containerd is used by Kubernetes via CRI

Key paths:
- `/etc/kubernetes/manifests` – control plane static pods
- `/var/lib/kubelet` – kubelet runtime state
- `/etc/containerd/config.toml` – container runtime config

---

## Fresh Cluster Setup (From Scratch)

### 1. Disable Swap (Fedora uses zram)

```bash
sudo mkdir -p /etc/systemd/zram-generator.conf.d
sudo tee /etc/systemd/zram-generator.conf.d/disable.conf <<EOF
[zram0]
zram-size = 0
EOF

sudo swapoff -a
sudo reboot
````

Verify:

```bash
swapon --summary
```

(Should produce no output)

---

### 2. Install & Configure containerd

```bash
sudo dnf install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit:

```toml
SystemdCgroup = true
```

Restart:

```bash
sudo systemctl enable --now containerd
```

Verify:

```bash
sudo ctr version
```

---

### 3. Kernel & Networking Prerequisites

```bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

### 4. Install Kubernetes Binaries (Pinned)

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubeadm kubelet kubectl
sudo dnf mark hold kubeadm kubelet kubectl
sudo systemctl enable --now kubelet
```

---

### 5. Initialize Cluster

```bash
sudo kubeadm init \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

---

### 6. Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait:

```bash
kubectl get pods -n kube-system -w
```

---

### 7. Allow Scheduling on Single Node

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

### 8. Install Metrics Server (with kubelet TLS workaround)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl -n kube-system patch deployment metrics-server \
  --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,Hostname"}
  ]'
```

Verify:

```bash
kubectl top nodes
```

---

## Daily Experiment Commands

```bash
kubectl get nodes
kubectl get pods -A
kubectl logs <pod>
kubectl exec -it <pod> -- sh
kubectl describe pod <pod>
```

Runtime-level debugging:

```bash
sudo crictl ps
sudo crictl pods
sudo journalctl -u kubelet -f
```

---

## Cluster Reset (Clean Rebuild)

```bash
sudo kubeadm reset -f
sudo rm -rf ~/.kube
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/etcd
sudo systemctl restart containerd kubelet
```

Then re-run kubeadm init.

---

## Certificate Maintenance

Check expiry:

```bash
sudo kubeadm certs check-expiration
```

Renew all certs:

```bash
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

---

## Common Troubleshooting

### Node NotReady

* CNI not installed or broken
* Check:

```bash
kubectl describe node <node>
kubectl -n kube-system logs -l k8s-app=calico-node
```

---

### kubelet CrashLoop

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

Common causes:

* swap enabled
* cgroup mismatch
* containerd not running

---

### metrics-server shows `0/1`

Cause:

* kubelet cert SAN mismatch

Fix:

* use `--kubelet-insecure-tls` (already applied in this setup)

---

## Firewalld Notes (Fedora)

For basic operation:

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --reload
```

---

## Upgrade Strategy (Recommended)

* Upgrade Kubernetes **once every 6–12 months**
* Control plane first
* Use kubeadm upgrade workflow
* Always snapshot `/etc/kubernetes` before upgrades

---

## Philosophy

This cluster is:

* **Intentionally close to production**
* **Not optimized for convenience**
* **Designed for learning internals**

Breaking and rebuilding the cluster is expected and encouraged.

---

## References

* [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
* [https://github.com/containerd/containerd](https://github.com/containerd/containerd)
* [https://docs.tigera.io/calico/latest](https://docs.tigera.io/calico/latest)
