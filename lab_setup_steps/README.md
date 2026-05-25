# Kubernetes Lab Cluster Setup on Ubuntu 22.04

A step-by-step guide to bootstrap a **3-node Kubernetes cluster** (1 control plane + 2 workers) on Ubuntu 22.04 using `kubeadm`, `containerd`, and `Flannel` as the CNI.

This is intended as a learning/lab setup. Each step includes verification commands so you can confirm things are healthy before moving on.

---

## Table of Contents

- [Cluster Topology](#cluster-topology)
- [Prerequisites](#prerequisites)
- [Step 1 — Disable Swap (all nodes)](#step-1--disable-swap-all-nodes)
- [Step 2 — Load Required Kernel Modules (all nodes)](#step-2--load-required-kernel-modules-all-nodes)
- [Step 3 — Configure Networking sysctl (all nodes)](#step-3--configure-networking-sysctl-all-nodes)
- [Step 4 — Install containerd (all nodes)](#step-4--install-containerd-all-nodes)
- [Step 5 — Install crictl (all nodes)](#step-5--install-crictl-all-nodes)
- [Step 6 — Install kubeadm, kubelet, kubectl (all nodes)](#step-6--install-kubeadm-kubelet-kubectl-all-nodes)
- [Step 7 — Initialize the Control Plane (control plane only)](#step-7--initialize-the-control-plane-control-plane-only)
- [Step 8 — Install Flannel CNI (control plane only)](#step-8--install-flannel-cni-control-plane-only)
- [Step 9 — Join the Worker Nodes (workers only)](#step-9--join-the-worker-nodes-workers-only)
- [Step 10 — Cluster Health Verification](#step-10--cluster-health-verification)
- [Step 11 — Smoke Test: Run an NGINX Pod](#step-11--smoke-test-run-an-nginx-pod)
- [Troubleshooting](#troubleshooting)

---

## Cluster Topology

| Role          | Count | Components                                                   |
| ------------- | :---: | ------------------------------------------------------------ |
| Control plane |   1   | `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd` |
| Worker        |   2   | `kubelet`, `kube-proxy`, application pods                    |

**Pod network CIDR:** `10.244.0.0/16` (Flannel default)

---

## Prerequisites

Before you begin, make sure you have:

- **3 × Ubuntu 22.04 LTS** machines (bare-metal or VMs)
- Each machine with at least **2 CPUs and 2 GB RAM** (control plane should ideally have 2+ GB)
- **Unique hostname, MAC address, and product_uuid** for every node
- **Full network connectivity** between all nodes (no firewall blocking the [required ports]
- **DNS server reachability and successful resolution from all the nodes(control plane and worker nodes)
- (https://kubernetes.io/docs/reference/networking/ports-and-protocols/))
- A user with **`sudo` privileges** on every node
- **Internet access** to pull packages and container images

> 💡 **Tip:** Set a clear hostname on each node before starting (e.g. `k8s-cp`, `k8s-worker-1`, `k8s-worker-2`) using `sudo hostnamectl set-hostname <name>`.

---

## Step 1 — Disable Swap (all nodes)

Kubelet refuses to run with swap enabled by default.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**Verify:**

```bash
free -h    # Swap line should show 0B
```

---

## Step 2 — Load Required Kernel Modules (all nodes)

`overlay` and `br_netfilter` are needed for container networking and iptables-based traffic filtering.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

# Persist across reboots
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

**Verify:**

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

---

## Step 3 — Configure Networking sysctl (all nodes)

Enable bridged traffic inspection by iptables and IP forwarding.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**Verify:**

```bash
cat /etc/sysctl.d/k8s.conf
sysctl net.ipv4.ip_forward            # should return 1
```

---

## Step 4 — Install containerd (all nodes)

`containerd` is the CRI runtime that actually runs the pods.

```bash
sudo apt-get update && sudo apt-get install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Switch to systemd cgroup driver (required by kubelet on systemd-based systems)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Verify:**

```bash
containerd --version
systemctl is-active containerd        # should print: active
systemctl is-enabled containerd       # should print: enabled
grep SystemdCgroup /etc/containerd/config.toml   # should show: SystemdCgroup = true
```

---

## Step 5 — Install crictl (all nodes)

`crictl` is the CLI for talking directly to the CRI runtime — extremely useful for troubleshooting.

```bash
VERSION="v1.35.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/${VERSION}/crictl-${VERSION}-linux-amd64.tar.gz
sudo tar -C /usr/local/bin -xzf crictl-${VERSION}-linux-amd64.tar.gz
rm crictl-${VERSION}-linux-amd64.tar.gz
```

**Verify:**

```bash
crictl --version
```

---

## Step 6 — Install kubeadm, kubelet, kubectl (all nodes)

```bash
# Tools to talk to the Kubernetes apt repo over HTTPS
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Create keyring directory
sudo mkdir -p /etc/apt/keyrings

# Add the Kubernetes signing key (v1.30 stream)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Register the Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install the three CLIs
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Pin so apt-get upgrade can't silently bump versions
sudo apt-mark hold kubelet kubeadm kubectl
```

**Verify:**

```bash
kubeadm version
kubelet --version
kubectl version --client
```

> 💡 To use a different Kubernetes minor version, swap `v1.30` in both the key URL and the repo line.

---

## Step 7 — Initialize the Control Plane (control plane only)

> ⚠️ Run the commands in this step **only on the control-plane node**.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

When this finishes, **copy the `kubeadm join …` command** printed at the end — you'll need it on the workers in Step 9.

Now configure `kubectl` for your user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Verify:**

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl describe node $(hostname) | grep -i podcidr
```

The control plane will show `NotReady` until a CNI is installed — that's expected and fixed in the next step.

---

## Step 8 — Install Flannel CNI (control plane only)

> ⚠️ Run **only on the control-plane node**.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Verify:**

```bash
kubectl get pods -n kube-flannel
kubectl get ds   -n kube-flannel
kubectl get nodes        # control-plane should now show Ready
```

---

## Step 9 — Join the Worker Nodes (workers only)

> ⚠️ Run **only on each worker node**.

If you lost the join command from Step 7, regenerate it from the **control plane**:

```bash
kubeadm token create --print-join-command
```

Then on each worker:

```bash
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify (from the control plane):**

```bash
kubectl get nodes -o wide
```

All three nodes should eventually show status `Ready`.

---

## Step 10 — Cluster Health Verification

Run these on the control plane to confirm everything is healthy:

```bash
# All 3 nodes registered and Ready?
kubectl get nodes -o wide

# Are the control-plane components healthy?
kubectl get pods -n kube-system

# One-line cluster summary
kubectl cluster-info

# Deeper component health
kubectl get componentstatuses

# Live event stream — catches errors that `get` hides
kubectl get events --all-namespaces --sort-by=.lastTimestamp
```

---

## Step 11 — Smoke Test: Run an NGINX Pod

```bash
kubectl run nginx --image=nginx
kubectl get pods -o wide
```

Wait until the pod's status is `Running`. To clean up:

```bash
kubectl delete pod nginx
```

🎉 **Congratulations — your Kubernetes lab cluster is ready!**

---

## Troubleshooting

| Symptom                                          | Likely cause / fix                                                                                  |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Nodes stuck `NotReady`                           | CNI not installed or pods crashing — check `kubectl get pods -n kube-flannel`                       |
| `kubeadm init` fails on swap check               | Re-run Step 1; confirm `free -h` shows `0B` swap                                                    |
| `kubeadm init` fails on cgroup driver mismatch   | Re-run Step 4; confirm `SystemdCgroup = true` in `/etc/containerd/config.toml`                      |
| Workers won't join (`x509` / token errors)       | Tokens expire after 24h — generate a fresh one with `kubeadm token create --print-join-command`     |
| Pods stuck in `ContainerCreating`                | Usually CNI-related — `kubectl describe pod <name>` and check kubelet logs (`journalctl -u kubelet`) |
| `kubectl` returns "connection refused"           | Ensure `$HOME/.kube/config` exists and the API server pod is running                                |

---

## References

- [Official kubeadm installation guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Flannel CNI](https://github.com/flannel-io/flannel)
- [containerd documentation](https://containerd.io/docs/)

---

## License

MIT — feel free to use, fork, and adapt for your own labs.
