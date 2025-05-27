# Master Node Setup Explanation

This document explains each command used in `setup.sh` to configure the Kubernetes master node.

- `hostnamectl`: Sets the hostname
- `swapoff` and `sed`: Disables swap, which Kubernetes requires
- `modprobe`: Loads kernel modules
- `sysctl`: Configures necessary networking parameters
- `containerd install/configure`: Installs and prepares container runtime
- `kubelet`, `kubeadm`, `kubectl`: Core Kubernetes components
- `kubeadm init`: Initializes the cluster
- `kubectl apply`: Applies the Flannel CNI plugin
