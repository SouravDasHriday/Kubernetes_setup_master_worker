# Worker Node Setup Explanation

This file explains each step used in the worker setup script.

- `hostnamectl`: Sets the node name
- `swapoff`, `sed`: Disables swap memory
- `modprobe`, `sysctl`: Enables networking and bridging
- `containerd install/configure`: Required container runtime
- `kubelet`, `kubeadm`, `kubectl`: Installs necessary Kubernetes components
- Final step reminds the user to join the master node using `kubeadm join`
