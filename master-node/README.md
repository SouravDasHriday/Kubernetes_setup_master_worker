🚀 Kubernetes Master Node Setup on Ubuntu (v1.28, kubeadm)

🔧 Step 1: Set Hostname (Optional but cleaner)
```bash
sudo hostnamectl set-hostname k8s-master
```
🔍 What it does:

This sets the system's hostname to k8s-master.
💡 Why:

Makes it easier to identify the node in your cluster.

Helpful when managing multiple nodes (e.g., k8s-worker-1, k8s-master).


🔕 Step 2: Disable Swap

Kubernetes requires swap to be off:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
🔍 What it does:

    swapoff -a: Disables swap immediately.

    sed ... /etc/fstab: Comments out swap entries in the file that controls swap on boot, so it's disabled permanently.

💡 Why:

Kubernetes requires RAM-only memory management.

Swap can cause unpredictable performance and violates Kubernetes resource scheduling logic.


📦 Step 3: Install Container Runtime (containerd)
```bash
sudo apt update
sudo apt install -y containerd
```
🔍 What it does:

Updates the package list and installs containerd, which is a container runtime.

💡 Why:

Kubernetes doesn’t run containers itself. It needs a container runtime (like Docker or containerd) to manage containers.

containerd is lightweight, fast, and officially supported by Kubernetes.


Generate default config:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.tom
```
🔍 What it does:

Prepares the config directory and writes the default configuration of containerd to config.toml.

💡 Why:

    You need this config to customize containerd behavior, especially the cgroup driver.


Use systemd as the cgroup driver (recommended for kubeadm):
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

```
🔍 What it does:

Changes the containerd setting to use systemd for cgroup (control group) management.

💡 Why:

Kubernetes and systemd need to use the same cgroup driver to avoid instability.

Systemd is the default init system in modern Linux distros (like Ubuntu), so aligning both improves compatibility.


Restart and enable:
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```
🔍 What it does:

Restarts containerd to apply changes.

Enables it to start at boot.

💡 Why:

Ensures containerd is running with the correct config every time the system boot


🌉 Step 4: Load Kernel Modules & sysctl Settings
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
🔍 What it does:

Configures and loads required kernel modules:

overlay: Required for container image layering.

br_netfilter: Enables iptables to see bridged traffic.

💡 Why:

Necessary for network communication between containers and between pods across nodes.


Enable networking settings:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```
🔍 What it does:

Configures system-wide networking rules to allow proper packet forwarding and firewalling for bridges (used by Kubernetes networking).

💡 Why:

Pods communicate across the cluster using bridges, and this sysctl tuning ensures traffic is allowed and properly routed.


🧩 Step 5: Install Kubernetes Components (kubelet, kubeadm, kubectl)
```bash
# Remove old Kubernetes sources
sudo rm -f /etc/apt/sources.list.d/kubernetes.list

# Set up new repo and keyring
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
🔍 What it does:

Clears any old repo list.

Adds Kubernetes's new repository signing key in a modern and secure format.

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```
🔍 What it does:

Adds the Kubernetes package repository (v1.28).

Updates the package list.


🧱 Install Kubernetes tools:

```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl  # Prevent auto-upgrade
```
🔍 What it does:

Installs core Kubernetes components:

kubelet: Runs and manages containers on this node.

kubeadm: Initializes and manages the cluster.

kubectl: CLI tool to interact with the cluster.

apt-mark hold: Prevents accidental upgrades that could break the setup.


 🚀 Step 6: Initialize the Master Node

Use a pod CIDR for your network plugin (Flannel = 192.168.0.0/16):
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
⚠️ Save the output join command (e.g. kubeadm join ...) — you'll need it to add worker nodes.

🔍 What it does:

Bootstraps a new Kubernetes cluster using kubeadm.

--pod-network-cidr: Specifies the IP range that the pod network plugin will use (Flannel uses 192.168.0.0/16).

💡 Why:

Essential to define the network plugin’s expected range during cluster initialization.


👤 Step 7: Set kubectl Access for Your User
Run this as your non-root user (or as root if no sudo user is used):
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
🔍 What it does:

Sets up kubectl so it can talk to the cluster using the admin credentials stored in admin.conf.

💡 Why:

Without this, you’ll get permission or context errors using kubectl.

🌐 Step 8: Apply Pod Network Add-on (Flannel)
```bash
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml

```
Wait 30–60 seconds and check pod/network status:

🔍 What it does:

Applies the Flannel network plugin configuration.

💡 Why:

Kubernetes doesn’t include a network plugin by default.

Flannel allows pods to communicate across nodes by creating a virtual overlay network.


🚫 Step 9: (Optional) Allow Workloads on Master Node
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
🔍 What it does:

Removes the taint that prevents pods from running on the master node.

💡 Why:

For single-node clusters, you often want to run pods on the master for testing.

Not needed if you have dedicated worker nodes.



✅ Final Checks
```bash
kubectl get nodes
kubectl get pods -n kube-flannel

```
See if the master node is Ready.

Check if Flannel pods are running successfully.




🧠 Summary

This setup configures your Ubuntu VM as a fully functional Kubernetes master node, including:

a) Swap off (Kubernetes requirement)

b) containerd setup (runtime)

c) Network configuration (sysctl, kernel modules)

d) Kubernetes components install

e) Cluster initialization

f) Networking with Flannel

g) Access with kubectl
