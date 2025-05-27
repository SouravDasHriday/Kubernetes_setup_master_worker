🧩 Goal

You’ll be preparing a worker node that will join an existing Kubernetes cluster managed by a master node you already set up.


🧱 Step 1: Set Hostname (Optional but Helpful)

```bash
sudo hostnamectl set-hostname k8s-worker-1
```

🔍 What it does:

Sets a readable hostname like k8s-worker-1 for easier identification.
💡 Why:

When managing multiple nodes, it's helpful to distinguish between master and worker nodes by name.

🔕 Step 2: Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

🔍 What it does:

    swapoff -a: Disables swap now.

    sed ...: Comments out the swap line in /etc/fstab to keep it off on boot.

💡 Why:

Kubernetes expects all available RAM to be used deterministically — swap disrupts that and is unsupported.


📦 Step 3: Install containerd

```bash
sudo apt update
sudo apt install -y containerd
```

🔍 What it does:

Installs containerd, the container runtime that Kubernetes will use to start and manage containers.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

🔍 What it does:

Generates the default containerd configuration and saves it in the correct directory.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

🔍 What it does:

Ensures containerd and kubelet both use the systemd cgroup driver (for consistency).
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

🔍 What it does:

Restarts containerd with the new config and enables it to start automatically on boot.

🌐 Step 4: Enable Network Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

🔍 What it does:

    Configures and loads kernel modules needed for container networking.

    overlay supports layered file systems for containers.

    br_netfilter enables iptables to see traffic from virtual bridges.

🔧 Step 5: Apply sysctl settings

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

🔍 What it does:

These settings:

    Enable forwarding of network packets between pods/nodes.

    Allow bridged network traffic (from containers) to be filtered through iptables.


    🔑 Step 6: Add Kubernetes Repo and Install Components

```bash
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

🔍 What it does:

    Cleans up old repo files if they exist.

    Adds the official Kubernetes signing key in a secure format.
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
```

🔍 What it does:

Adds the Kubernetes v1.28 package repository and updates APT.

```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
🔍 What it does:

    Installs the core Kubernetes components.

        kubelet: Agent running on all nodes, starts pods/containers.

        kubeadm: Used to join the cluster.

        kubectl: Optional on workers, but useful for debugging.

    Holds the versions to avoid unwanted upgrades.

🔗 Step 7: Join the Cluster (The Key Step)

You'll get this command from the master node, right after you run kubeadm init:

kubeadm join <MASTER-IP>:6443 --token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>

🔍 What it does:

    Connects the worker to the master via secure communication.

    Uses the provided token and hash to authenticate and verify the cluster.

💡 How to get this command again:

If you missed it or lost it, go to your master node and run:

```bash
kubeadm token create --print-join-command
```

That will regenerate the join command for you.

✅ Step 8: Verify Join (From Master Node)

Once joined, go back to the master node and run:

```bash
kubectl get nodes
```

🔍 What it does:

Lists all nodes in the cluster.

    Master and all joined workers should appear.

    Status should be Ready once networking is up.

🧠 Summary: What Happens Internally
| Action                       | Purpose                                                |
| ---------------------------- | ------------------------------------------------------ |
| Disable swap                 | Required by Kubernetes for consistent memory handling  |
| Install containerd           | Container runtime used to run pods and containers      |
| Load kernel modules + sysctl | Enable networking between pods and across nodes        |
| Add Kubernetes repo          | Get up-to-date and signed packages                     |
| Install kubelet/kubeadm      | Required for the node to function and join the cluster |
| `kubeadm join`               | Registers the node with the master, securely           |
