# Kubernetes (K8s) Step-by-Step Installation Guide on Ubuntu from Scratch (Advanced Edition)
![k8s](https://imgur.com/bKUeyKX.png)
Certainly! Here's an **advanced and fully explained** step-by-step guide to installing Kubernetes (K8s) on Ubuntu from scratch as of January 17, 2026 (Kubernetes v1.35).
This version uses the modern, recommended approach:
- `containerd` as the container runtime (Docker is deprecated in Kubernetes)
- Official `pkgs.k8s.io` repositories
- Calico with the Tigera operator as the CNI (network plugin)
Every command (or group of commands) is followed by a clear, beginner-friendly explanation of **what it does** and **why it's needed**. The guide is structured to work 100% on a multi-node setup (1 master/control-plane node + 1 or more worker nodes).

**Prerequisites** (All Nodes)
1. Ubuntu 22.04 LTS or 24.04 LTS (clean install recommended)
2. At least 2 nodes: 1 master (minimum 2 CPUs, 4 GB RAM) + 1 or more workers (minimum 2 CPUs, 2 GB RAM each)
3. Nodes can reach each other over the network (use static IPs if possible)
4. Unique hostnames on each node (e.g., `master-node`, `worker-1`)
5. sudo/root access
6. Internet access for downloading packages

**Recommended**: Set hostnames before starting
```bash
# On master node
sudo hostnamectl set-hostname master-node
# On each worker node (use different names)
sudo hostnamectl set-hostname worker-1
```

**Step 1: Common Setup on ALL Nodes** (Master + Workers)

**1.1 Update system and install basic tools**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg
```
**Explanation**:
`apt update` refreshes the list of available packages. `apt upgrade` installs the latest security patches and updates. This prevents conflicts with old packages. The extra tools (`curl`, `gpg`, etc.) are needed later to securely add Kubernetes repositories.

**1.2 Disable swap**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
**Explanation**:
Kubernetes does **not** work well with swap memory. Swap can cause unpredictable performance and interfere with Kubernetes' memory management (e.g., container OOM killing). `swapoff -a` disables it immediately. Editing `/etc/fstab` comments out the swap line so it stays disabled after reboot.

**1.3 Load kernel modules and configure networking parameters**
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
- `overlay` and `br_netfilter` are kernel modules required for container storage and networking (containerd uses overlay filesystem, and Kubernetes needs bridge networking).
- `modprobe` loads them now.
- The `sysctl` settings enable IP forwarding (needed for pods to communicate across nodes) and allow iptables rules to work on bridged traffic. `sysctl --system` applies them immediately and permanently.

**1.4 Install and configure containerd**
```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```
**Explanation**:
containerd is the container runtime that actually runs your pods. Kubernetes talks to it via CRI (Container Runtime Interface).
We generate its default config, then change `SystemdCgroup = true` so Kubernetes can properly manage cgroups (resource limits) using systemd (the default on Ubuntu). Finally, restart and enable it to start on boot.

**1.5 Add Kubernetes repository and install tools**
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
This adds the official Kubernetes package repository for v1.35. The GPG key ensures packages are authentic.
`kubeadm` = cluster bootstrapping tool
`kubelet` = agent that runs on every node to manage pods
`kubectl` = command-line tool to control the cluster
`apt-mark hold` prevents accidental upgrades that could break the cluster. Enabling kubelet starts it now and on boot.


**Step 2: Initialize the Master (Control Plane) Node**
Run only on the master node:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=MASTER_IP
```
Replace `MASTER_IP` with the master's actual IP. If the master has only one network interface, you can omit `--apiserver-advertise-address` (it will auto-detect).
**Explanation**:
`kubeadm init` sets up the control plane components: API server, etcd (database), controller manager, scheduler, and core DNS.
`--pod-network-cidr` reserves a IP range for pod networking (Calico will use this exact range by default).
After it finishes, it prints a `kubeadm join` command — **copy and save it** (you'll need it for workers).


**Step 3: Configure kubectl on the Master**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
**Explanation**:
`kubectl` needs a config file to know how to talk to your cluster's API server. This copies the admin credentials and fixes permissions so your regular user can use `kubectl`.


**Step 4: Install Calico Network Plugin** (on Master)
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/custom-resources.yaml
```
**Explanation**:
Pods can't communicate without a CNI (Container Network Interface) plugin. Calico provides networking and network policy.
The first command installs the Tigera operator (manages Calico). The second applies the default configuration, which uses the exact pod CIDR we specified (`192.168.0.0/16`).
Wait for Calico to be ready:
```bash
watch kubectl get tigerastatus
```
All statuses should eventually show `Available`.


**Step 5: Join Worker Nodes**
On **each worker node**, run the `kubeadm join` command you saved from Step 2 (it will look like):
```bash
sudo kubeadm join 192.168.1.10:6443 --token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
**Explanation**:
This registers the worker with the master, installs the necessary components, and pulls certificates securely. The token and hash ensure only authorized nodes can join.


**Step 6: Verify the Cluster** (on Master)
```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pods -n calico-system
```
**Explanation**:
- `kubectl get nodes` should show all nodes as `Ready`.
- The kube-system and calico-system pods should all be `Running`.

- 
**Step 7: Test with a Basic Pod Creation**
Now let's create a real pod to prove everything works end-to-end.
On the master node:
```bash
# Create a simple Nginx pod
kubectl create deployment nginx-test --image=nginx --replicas=2
# Watch the pods come up
kubectl get pods -w
# Check details
kubectl get pods -o wide
kubectl get deployments
kubectl describe deployment nginx-test
# (Optional) Expose it to test access
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get services
# Note the NodePort (e.g., 3XXXX), then access http://WORKER_IP:NODEPORT from your browser
# Clean up when done
kubectl delete deployment nginx-test
kubectl delete service nginx-test
```
**Explanation of the test**:
- `kubectl create deployment` creates a Deployment object that manages 2 replicas of an Nginx container.
- The scheduler places the pods on available nodes (usually workers).
- When pods are `Running`, it means: control plane → kubelet → containerd → networking all work.
- `get pods -o wide` shows which node each pod runs on.
- Exposing as NodePort lets you access the app from outside to confirm networking.
Congratulations! You now have a fully functional Kubernetes v1.35 cluster.
**Notes**:
- For production: use multiple control-plane nodes (HA), enable proper firewall rules, and add security hardening.
- Always refer to official docs for the absolute latest manifests/versions.
## By [Sagar Malviya](https://www.github.com/sagarmalviyaa) (Updated January 17, 2026) give this in markdown file
