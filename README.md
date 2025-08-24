# Kubernetes Installation Guide (Ubuntu) 🧭

> **Scope:** This guide installs a Kubernetes cluster using `kubeadm` and `containerd` on **three Ubuntu servers**. It covers two topologies:
>
> 1. **Single control plane:** 1 master + 2 workers
> 2. **HA control plane:** 3 control-plane nodes (also usable as workers)

> **Tested:** Ubuntu 22.04/24.04, Kubernetes v1.30, containerd.
> **Networking:** Calico (v3.25.0).

---

## 0) Hostnames & /etc/hosts (on **all** servers)

```bash
sudo vi /etc/hosts
```

Add the following entries (adjust IPs to your environment):

```
192.168.1.111 k8s-master-1
192.168.1.112 k8s-master-2
192.168.1.113 k8s-master-3
```

> 📝 Ensure each node's hostname matches (e.g., `hostnamectl set-hostname k8s-master-1`).

---

## 1) Update system (on **all** servers)

```bash
sudo apt update -y && sudo apt upgrade -y
```

---

## 2) Create and switch to devops user (optional)

```bash
sudo adduser devops
su - devops
cd /home/devops
```

> You can run the rest as `devops` with `sudo`.

---

## 3) Disable swap (on **all** servers)

```bash
sudo swapoff -a
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```

---

## 4) Load kernel modules (on **all** servers)

Create modules file:

```bash
sudo tee /etc/modules-load.d/containerd.conf <<'EOF'
overlay
br_netfilter
EOF
```

Load modules now:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 5) Sysctl for networking (on **all** servers)

```bash
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf

sudo sysctl --system
```

---

## 6) Install prerequisites & Docker repo (on **all** servers)

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

---

## 7) Install containerd (on **all** servers)

```bash
sudo apt update -y
sudo apt install -y containerd.io
```

Generate default config and switch to systemd cgroups:

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 8) Add Kubernetes apt repo (on **all** servers)

```bash
sudo mkdir -p /etc/apt/keyrings

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

---

## 9) Install Kubernetes components (on **all** servers)

```bash
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> 🔒 Holding prevents accidental upgrades that can break your cluster.

---

## 10) (Optional) Reset a misconfigured cluster

If you need to reset after an attempted init:

```bash
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes/manifests/*
```

---

# Cluster Topology Options

## Option A — 1 Control-Plane (master) + 2 Workers

**Run on `k8s-master-1`:**

```bash
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

**Join worker nodes (`k8s-master-2` and `k8s-master-3` acting as workers):**

Use the join command printed by `kubeadm init`, or re-generate it with:

```bash
kubeadm token create --print-join-command
```

Then run it on each worker, e.g.:

```bash
sudo kubeadm join 192.168.1.111:6443 --token <YOUR_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<YOUR_CA_CERT_HASH>
```

---

## Option B — HA Control Plane (3 Masters)

**Run on `k8s-master-1`:**

```bash
sudo kubeadm init --control-plane-endpoint "192.168.1.111:6443" --upload-certs

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

> 📌 `--upload-certs` prints a `--certificate-key`. Save it to join other control-plane nodes.

**Run on `k8s-master-2` and `k8s-master-3`:**

```bash
sudo kubeadm join 192.168.1.111:6443 \
  --token <YOUR_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<YOUR_CA_CERT_HASH> \
  --control-plane \
  --certificate-key <YOUR_CERTIFICATE_KEY>

# (Optional) Setup kubectl for convenience on each master
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 💡 In HA setups you typically front the control plane with a virtual IP / load balancer. Here we reuse `192.168.1.111:6443` as the endpoint (adjust to your LB VIP if used).

---

## 11) Post‑install checks

```bash
# On a control-plane node
kubectl get nodes -o wide
kubectl get pods -A
```

Wait until all core components (kube-system) are `Running`/`Ready`.

---

## 12) Useful extras (optional)

Bash completion & kubectl alias:

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

---

## Troubleshooting

* **CNI not ready / nodes not Ready:** Confirm Calico applied successfully and no network blocks by firewall.
* **kubeadm join fails (token/CA hash):** Recreate join command with `kubeadm token create --print-join-command` on a control-plane node.
* **Swap re-enabled after reboot:** Re-check `/etc/fstab` and ensure swap line is commented.
* **containerd cgroup mismatch:** Verify `SystemdCgroup = true` in `/etc/containerd/config.toml` and restart containerd.

---

## Version Pins

* Kubernetes repo in this guide targets **v1.30**. If you pin to a different minor, adjust the repo URL accordingly.
* Calico used here is **v3.25.0**; you may choose a newer version compatible with your K8s minor.

---

## License

MIT
