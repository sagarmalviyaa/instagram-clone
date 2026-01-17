# Kubernetes (K8s) Step-by-Step Installation Guide on Ubuntu from Scratch (Advanced Edition)

![k8s](https://media.licdn.com/dms/image/v2/C4E12AQGT9MMqFNOAMA/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1594108349733?e=2147483647&v=beta&t=U4VyScSv3B1aOpF8XFkOqdxF3BmGdZ670a-l8fOqTHM)

Certainly! Below is an **advanced, production-aligned, and fully explained** step-by-step guide to installing **Kubernetes v1.35** on **Ubuntu 22.04 / 24.04 LTS** as of **January 17, 2026**.

This guide uses:

* **containerd** as the container runtime (Docker is deprecated)
* Official **pkgs.k8s.io** repositories
* **Calico (classic manifest)** as the CNI (simpler and more stable for kubeadm clusters)

Every command is followed by a clear explanation of **what it does** and **why it is required**. The guide is written for a **multi-node cluster**:

* 1 Control Plane (Master)
* 1 or more Worker Nodes

---

## Prerequisites (All Nodes)

1. Ubuntu **22.04 LTS** or **24.04 LTS** (clean install recommended)
2. Minimum **2 nodes**

   * Master: 2 CPU, 4 GB RAM
   * Worker: 2 CPU, 2 GB RAM
3. All nodes can reach each other over the network
4. Static IPs recommended
5. Unique hostnames on each node
6. sudo / root access
7. Internet connectivity

---

## Recommended: Set Hostnames

```bash
# On master node
sudo hostnamectl set-hostname master-node
echo 'export PS1="[$(hostname --static)@\w]\\$ "' >> ~/.bashrc && \
source ~/.bashrc

# On worker nodes
sudo hostnamectl set-hostname worker-1
echo 'export PS1="[$(hostname --static)@\w]\\$ "' >> ~/.bashrc && \
source ~/.bashrc
```

**Explanation**:
Kubernetes strongly relies on unique node identities. Setting hostnames early prevents confusion during node registration and troubleshooting.

---

## Step 1: Common Setup on ALL Nodes (Master + Workers)

### 1.1 Update system and install base utilities

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

**Explanation**:

* Updates all system packages and security patches
* Installs tools required to securely download and verify Kubernetes packages

---

### 1.2 Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**Explanation**:
Kubernetes depends on precise memory management. Swap can cause unpredictable pod eviction and resource pressure. Kubernetes **will not start** if swap is enabled.

---

### 1.3 Load Kernel Modules and Configure Networking

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

**Explanation**:

* `overlay`: Required for container filesystem layers
* `br_netfilter`: Allows iptables to inspect bridged traffic
* IP forwarding enables pod-to-pod and pod-to-node communication

---

### 1.4 Install and Configure containerd

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Explanation**:
containerd is the runtime that actually launches containers. Enabling `SystemdCgroup` ensures compatibility with kubelet and systemd-based resource control.

---

### 1.5 Install Kubernetes Components

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

**Explanation**:

* `kubeadm`: Bootstraps the cluster
* `kubelet`: Node-level agent that runs pods
* `kubectl`: CLI to manage Kubernetes
* Version hold prevents accidental upgrades

---

## Step 2: Initialize the Control Plane (Master Node Only)

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=MASTER_IP
```

**Explanation**:

* Initializes etcd, API server, scheduler, and controller-manager
* `pod-network-cidr` must match the CNI configuration
* Save the `kubeadm join` command printed at the end

---

## Step 3: Configure kubectl Access (Master Node)

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Explanation**:
This grants your normal user administrative access to the cluster using `kubectl`.

---

## Step 4: Install Calico Network Plugin (Stable Method)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

**Explanation**:

* Installs Calico using the **classic manifest**, which is widely used with kubeadm
* Automatically configures networking, IP pools, and controllers
* Uses the same default pod CIDR: `192.168.0.0/16`
* Avoids Tigera operator issues commonly seen in labs and fresh clusters

Verify Calico:

```bash
kubectl get pods -n kube-system
```

All Calico-related pods should reach `Running` state.

---

## Step 5: Join Worker Nodes

On each worker node, run the `kubeadm join` command from Step 2:

```bash
sudo kubeadm join <MASTER_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Explanation**:
This securely joins the worker node to the control plane and registers it with the cluster.

---

## Step 6: Verify the Cluster (Master Node)

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

**Expected Result**:

* All nodes should be `Ready`
* Core system pods should be `Running`

---

## Step 7: Test the Cluster with a Sample Application

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl get pods -o wide
kubectl expose deployment nginx-test --type=NodePort --port=80
kubectl get services
```

Access from browser:

```
http://<WORKER_IP>:<NODE_PORT>
```

Cleanup:

```bash
kubectl delete deployment nginx-test
kubectl delete service nginx-test
```

---

## Final Notes

* This cluster is suitable for **learning, labs, and staging**
* For production: enable HA control plane, firewall rules, monitoring, and RBAC hardening
* Always match Kubernetes and CNI versions carefully

---

### By **Sagar Malviya**

🔗 [https://www.github.com/sagarmalviyaa](https://www.github.com/sagarmalviyaa)

**Last Updated: January 17, 2026**
