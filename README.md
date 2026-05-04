# kubernete_cluster_setup_on_EC2
Launch an EC2 Instance with t2.medium
update the 
# Master Node
### Change the Hostname of Master Node
'sudo hostnamectl set-hostname control-plane

sudo apt-get update 
Master Node Setup
Update the package index
Bash
sudo apt-get update 
Update packages required for HTTPS package repository access
Bash
sudo apt-get install -y 
sudo apt install ca-certificates curl software-properties-common gnupg lsb-release
Allow forwarding IPv4
Load the br_netfilter module with the following commands:

Bash
# Load br_netfilter module
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
Disable swap partition
Bash
sudo swapoff -a
Configure sysctl params
These parameters are required by setup and persist across reboots:

Bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
Install containerd
Using the DEB package distributed by Docker:

Bash
# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up the repository
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io
Note: This is only one way of installing containerd. Please refer to the containerd docs.

Configure the systemd cgroup driver
This is required to mitigate the instability of having two cgroup managers.

Bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
Install kubeadm, kubectl, and kubelet
Bash
# Add the public signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes release repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update and install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl
Initialize the control-plane node
Bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 

# Setup local kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Verify and Create Calico Network
Bash
kubectl get componentstatuses
kubectl get nodes

# Create the Calico network plugin for pod networking
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
Worker Node Setup
The steps for the Worker Node are largely similar to the Master Node for system preparation.

1. System Preparation
Run the updates, load modules, and configure sysctl exactly as shown in the Master Node section.

2. Disable swap
Bash
sudo swapoff -a
3. Join the Cluster
Once the master is initialized, run your join command (example below):

Bash
kubeadm join 10.0.0.100:6443 --token mtcv1t.pvyu0ij061oake7t --discovery-token-ca-cert-hash sha256:991981f37a96591e8e4fe57ce761ab7b7832a8a90c76e612234fc1c8b9fcbb55 
Kubectl Autocomplete
Enable auto-completion in Linux for a smoother experience:

Bash
sudo apt-get install bash-completion
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
