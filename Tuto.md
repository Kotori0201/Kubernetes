# **Setup Virtual Machine**

## **Prerequisites**

- **Ubuntu 24.04**: Make sure you have a clean, up-to-date installation of Ubuntu 24.04 on all the machines you intend to include in your cluster.
- **Minimum Resources**: Each machine should have at least 2 CPU cores, 2GB of RAM, and enough disk space to accommodate the Kubernetes components.
- **Static IP Addresses**: Assign static IP address to each node in the cluster to ensure stability and consistency.
- **SSH Access**: Ensure you have SSH access to all the nodes from the machine you will be using to install the cluster.
- **Privileged user** with sudo rights.
- **Stable** Internet Connection.

## **Setup Virtual Machine**

### Config Host Network Manager

> On screen Oracle VM VirtualBox Manager, File &rarr; Host Network Manager

**Properties &rarr; Adapter &rarr; Select `Configure Adapter Manually`**:

- IPv4 Address: 10.10.10.1
- IPv4 Network Mask: 255.255.255.0

**Properties -> DHCP Server -> Select `Enable Server`**:

- Server Address: 10.10.10.1
- Server Mask: 255.255.255.0
- Lower Address Bound: 10.10.10.2
- Upper Address Bound: 10.10.10.254

**After configuration, Click `Apply` to done.**

## Create Virtual Machine

- Memory size: 4GB of RAM.
- File size: 100 GB.
- Hard disk file type: VDI.
- Storage on physical hard disk: Fixed size.

## Setting Virtual Machine

### System

- Motherboard:
  - Base Memory: 4096 MB.
  - Boot Order: Only select:
      1. Hard Disk.
      2. Optical.
  - Pointing Device: PS/2 Mouse.
  - Extended Features: Select all options.

### Processor

- Processor(s): 2 CPU.
- Extended Features: Select **Enable PAE/NX.

### Acceleration

- Paravirtualization Interface: KVM.

### Storage

>Controller: IDE -> Choose disk file: ubuntu-24.04.1-live-server-amd64.iso

### Network

- Adapter 1: Select 'Enable Network Adapter' -> Attached to: NAT

- Adapter 2: Select 'Enable Network Adapter' -> Attached to: Host-only Adapter

**After configuration, click OK.**

## Run and Boot Virtual Machine

- Language: English
- Keyboard configuration.
- Choose the base for the installation: Ubuntu Server.
- Network configuration.
- Guided storage configuration: Custom storage layout -> Choose path Mount.
- Profile configuration.
- Installing system.

---

# Setup with kubeadm 1 master && 3 worker

## 1 Set hostname of Each Node

Use hostnamectl command to set hostname on each node, example is shown below:

```bash
 hostnamectl set-hostname master-node-01
 hostnamectl set-hostname worker-node-01
 hostnamectl set-hostname worker-node-02
 hostnamectl set-hostname worker-node-03
 ```

sudo vim /etc/hosts

```bash
 172.10.0.2 master-node-01 
 172.10.0.3 worker-node-01
 172.10.0.4 worker-node-02
 172.10.0.5 worker-node-03
```

## 2 Disable swap and Add Kernel Modules

Disable swap and add following kernel module on all the nodes ( master + worker nodes).

To disable swap, edit /etc/fstab file and comment out the line which includes entry either swap partition or swap file.

- sudo vi /etc/fstab

![](https://www.linuxtechi.com/wp-content/uploads/2020/06/Swap-disable-Ubuntu-20-04.png?ezimgfmt=ng:webp/ngcb22)

- sudo swapoff -a

```bash
$ sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilter

$ sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

$ sudo sysctl --system
```

## 3 Install Containerd Runtime on All Nodes

Use hostnamectl command to set hostname on each node, example is
shown below:

### cach 1

[link huong dan](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

### cach 2

```bash
apt search containerd | grep containerd
apt install containerd
```

### How to configure Containerd

With everything installed, we can now configure Containerd. Create a new directory to house the Containerd configurations with:

- sudo mkdir /etc/containerd

Create the configurations with:

- containerd config default | sudo tee /etc/containerd/config.toml

Enable SystemdCgroup with the command:

- sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

Download the required systemd file with:

- sudo curl -L <https://raw.githubusercontent.com/containerd/containerd/main/containerd.service> -o /etc/systemd/system/containerd.service

Reload the systemd daemon with:

- sudo systemctl daemon-reload

- sudo systemctl enable --now containerd

You can verify everything is running with the command:

- sudo systemctl status containerd

**You should see output similar to this:

![status](https://raw.github.sec.samsung.net/vietanh-pham/k8s-for-beginner/main/container-d-status.jpg?token=GHSAT0AAAAAAAACUGVOZP6YY26BNHEAGUBAZYHKCPQ)

## 4 Install Kubernetes tools (kubeadm, kubectl, kubelet)

Before you begin

- A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
- 2 GB or more of RAM per machine (any less will leave little room for your apps).
- 2 CPUs or more for control plane machines.
Full network connectivity between all machines in the cluster (public or private network is fine).

Note:
Docker Engine does not implement the CRI which is a requirement for a container runtime to work with Kubernetes. For that reason, an additional service cri-dockerd has to be installed. cri-dockerd is a project based on the legacy built-in Docker Engine support that was removed from the kubelet in version 1.24.

The tables below include the known endpoints for supported operating systems:

```
Runtime Path to Unix domain socket
containerd     | unix:///var/run/containerd/containerd.sock
CRI-O          | unix:///var/run/crio/crio.sock
Docker Engine  | unix:///var/run/cri-dockerd.sock
(cri-dockerd) 
```

These instructions are for Kubernetes v1.31.

Update the apt package index and install packages needed to use the Kubernetes apt repository:

sudo apt-get update

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
### Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
### If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### Note

In releases older than Debian 12 and Ubuntu 22.04, directory /etc/apt/keyrings does not exist by default, and it should be created before the curl command.
Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.31; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).

#### This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

(Optional) Enable the kubelet service before running kubeadm:
```sudo systemctl enable --now kubelet```

The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

Configuring a cgroup driver
Both the container runtime and the kubelet have a property called "cgroup driver", which is important for the management of cgroups on Linux machines.

# 3. pull images

sudo mkdir -p -m 755 /etc/apt/keyrings

config image registry.k8s.io 3.8 --> 3.9 / 3.10

```bash
sudo kubeadm config images list
for i in $(sudo kubeadm config images list); do sudo ctr -n k8s.io images pull $i -k; done
sudo kubeadm config images pull
```

xong nhớ check image version o /etc/containerd/config.toml, search image_sanbox có đúng version 3.10 hay ko

### init

kubeadm config print init-defaults | cat > kubeadm_config.yaml

### template kubeadm init

```bash
kubeadm init 
 --apiserver-advertise-address=172.10.0.2 
 --apiserver-cert-extra-sans=172.10.0.2 
 --node-name  master-node-01 or --control-plane-endpoint=master-node-01
 --pod-network-cidr=10.10.0.0/16
```

```bash
kubeadm init --apiserver-advertise-address=192.168.56.10 --control-plane-endpoint=k8s-master --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/containerd/containerd.sock --apiserver-cert-extra-sans=192.168.56.30 --service-cidr=192.168.56.30/24
```

After the initialization process completes, you’ll see a message containing a ‘kubeadm join’ command. Save this command; we’ll use it later to add worker nodes to the cluster.

In order to interact with cluster as a regular user, let’s execute the following commands, these commands are already there in output just copy paste them.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubeadm token create --print-join-command

kubeadm join master-node-01:6443 --token 2u9hs6.u1xzxi6uk4uu9lov --discovery-token-ca-cert-hash sha256:f3be9b23b8adb205134a2992affca3078c0f1828037aab643602c19a9c0a8ba7 
```

<<<<<<< HEAD
### Add Worker Nodes to Kubernetes Cluster
=======
Add Worker Nodes to Kubernetes Cluster
>>>>>>> 88c93f17bfadd88cedfb3772a99d5b4d69df60ed

### compare stacked etcd vs external etcd