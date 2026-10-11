# Kubernetes Cluster Installation with kubeadm

## Installation Overview

Kubernetes cluster installation consists of two separate processes.

| Guide                      | Purpose                                                                                                  |
| -------------------------- | -------------------------------------------------------------------------------------------------------- |
| Part 1: Prepare the Nodes  | Prepare Linux nodes and install containerd, kubeadm, kubelet, and kubectl.                               |
| Part 2: Create the Cluster | Initialize the control plane, install a Container Network Interface (CNI) plugin, and join worker nodes. |

This documentation follows these two stages to build a Kubernetes cluster with one control-plane node and two worker nodes.

# Part 1: Prepare the Nodes

## 1. Lab Architecture

The lab will consist of three Linux virtual machines.

```text
                    Kubernetes Cluster
                           |
                 +---------v---------+
                 |   Control Plane   |
                 |                   |
                 | kube-apiserver    |
                 | etcd              |
                 | scheduler         |
                 | controller-manager|
                 | kubelet           |
                 | containerd        |
                 +---------+---------+
                           |
               +-----------+-----------+
               |                       |
       +-------v--------+      +-------v--------+
       |   Worker 01    |      |   Worker 02    |
       |                |      |                |
       | kubelet        |      | kubelet        |
       | containerd     |      | containerd     |
       | CNI networking |      | CNI networking |
       +----------------+      +----------------+
```

### 1.1 Node Configuration

The following IP addresses and hostnames are examples. Replace them with the actual values assigned to your virtual machines.

| Node          | Hostname       | Example IP      | Role                                     |
| ------------- | -------------- | --------------- | ---------------------------------------- |
| Control Plane | `k8s-master`   | `192.168.10.10` | Runs Kubernetes control-plane components |
| Worker 01     | `k8s-worker01` | `192.168.10.11` | Runs application workloads               |
| Worker 02     | `k8s-worker02` | `192.168.10.12` | Runs application workloads               |

## 2. Prerequisites

Before installing Kubernetes components, verify that all three machines meet the following requirements.

| Requirement            | Control Plane | Worker Nodes  |
| ---------------------- | ------------- | ------------- |
| Linux operating system | Required      | Required      |
| CPU                    | 2 minimum     | 2 recommended |
| RAM                    | 2 GB minimum  | 2 GB minimum  |
| Unique hostname        | Required      | Required      |
| Unique MAC address     | Required      | Required      |
| Unique product UUID    | Required      | Required      |
| Network connectivity   | Required      | Required      |

**Lab recommendation:** Allocate at least 4 GB RAM per node for a more comfortable learning environment.

### 2.1 Verify the Operating System

Run on all nodes:

```bash
cat /etc/os-release
```

Check the Linux kernel version:

```bash
uname -r
```

Kubernetes relies on Linux kernel features for container execution, networking, and other cluster operations.

### 2.2 Verify the Product UUID

Run on all nodes:

```bash
cat /sys/class/dmi/id/product_uuid
```

> [!IMPORTANT] 
Compare the output from all three machines and ensure that each node has a unique product UUID. 
Unique node identifiers help Kubernetes distinguish machines and prevent node registration conflicts.

## 3. Configure Hostnames

Assign a unique hostname to each node.

### 3.1 Control Plane

Run on the control-plane node:

```bash
hostnamectl set-hostname k8s-master
```

### 3.2 Worker 01

Run on the first worker node:

```bash
hostnamectl set-hostname k8s-worker01
```

### 3.3 Worker 02

Run on the second worker node:

```bash
hostnamectl set-hostname k8s-worker02
```

### 3.4 Verify the Hostname

Run on each node:

```bash
hostname
```

Expected output:

* Control Plane: `k8s-master`
* Worker 01: `k8s-worker01`
* Worker 02: `k8s-worker02`

## 4. Configure Network Connectivity

All three nodes must be able to communicate with each other over the network.

### 4.1 Configure Hostname Resolution

On **all three nodes**, open the `/etc/hosts` file:

```bash
vim /etc/hosts
```

Add the following entries, adjusting the IP addresses if necessary:

```text
192.168.10.10 k8s-master
192.168.10.11 k8s-worker01
192.168.10.12 k8s-worker02
```

Save the file.

### 4.2 Verify Name Resolution and Connectivity

Run the following commands from each node:

```bash
ping -c 3 k8s-master
ping -c 3 k8s-worker01
ping -c 3 k8s-worker02
```

Verify the node's routing table:

```bash
ip route
```

Confirm that the network interface used for Kubernetes communication has the correct IP address and routing configuration.

> [!IMPORTANT] 
If a machine has multiple network interfaces, ensure Kubernetes uses the appropriate interface for node-to-node communication. Successful ping tests are useful but do not replace verification of the required Kubernetes ports and firewall rules.

---

## 5. Disable Swap

For this lab, swap will be disabled on all nodes. This is the conventional kubeadm configuration and avoids the default kubelet behavior of refusing to start when swap is enabled.

### 5.1 Check Swap Status

Run on all nodes:

```bash
swapon --show
```

If swap is enabled, the command displays the active swap devices or files.

### 5.2 Disable Swap Temporarily

```bash
sudo swapoff -a
```

This disables active swap until it is enabled again or the machine reboots with swap configured.

### 5.3 Disable Swap Permanently

Open the filesystem configuration file:

```bash
sudo vi /etc/fstab
```

Locate the swap entry and comment it out. For example:

```text
# /swap.img none swap sw 0 0
```

The actual entry may differ depending on the operating system. Do not comment out unrelated filesystem entries.

### 5.4 Verify Swap Status

```bash
swapon --show
free -h
```

Expected result: `swapon --show` displays no active swap devices, and the `free -h` output shows zero swap usage and total swap capacity if no other swap device is configured.

## 6. Load Required Kernel Modules

Kubernetes container networking requires kernel support for operations such as overlay filesystems and bridge traffic processing.

Perform the following steps on **all three nodes**.

### 6.1 Load the Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

* `overlay`: Enables the overlay filesystem used by many container images and writable container layers.
* `br_netfilter`: Allows bridged network traffic to be processed by applicable Netfilter rules.

### 6.2 Make the Modules Persistent

Create a module configuration file:

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

The configuration ensures that these modules are loaded automatically during subsequent boots.

### 6.3 Verify the Modules

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

Both modules should appear in the output.

## 7. Configure Kubernetes Networking Parameters

Kubernetes networking requires IP forwarding and appropriate Netfilter settings.

Perform these steps on **all three nodes**.

### 7.1 Create the Sysctl Configuration

Create the configuration file:

```bash
vim /etc/sysctl.d/k8s.conf
```

Add the following settings:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

### 7.2 Apply the Settings

```bash
sysctl --system
```

This command reloads applicable system-wide sysctl configuration files.

### 7.3 Verify IP Forwarding

```bash
sysctl net.ipv4.ip_forward
```

Expected output:

```text
net.ipv4.ip_forward = 1
```

Verify the bridge Netfilter parameters as well:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

Expected values for both parameters:

```text
= 1
```
---

## 8. Install and Configure containerd

Containerd is the container runtime that Kubernetes uses to manage container images and containers. Perform the following steps on **all three nodes**.

### 8.1 Install Required Repository Management Tools

```bash
sudo dnf install -y dnf-plugins-core
```

### 8.2 Configure the Docker Repository

Add the official Docker repository for RHEL:

```bash
sudo dnf config-manager --add-repo \
https://download.docker.com/linux/rhel/docker-ce.repo
```

Verify the repository:

```bash
sudo dnf repolist
```

### 8.3 Install containerd

```bash
sudo dnf install -y containerd.io
```

Verify the installed version:

```bash
containerd --version
rpm -q containerd.io
```

### 8.4 Generate the containerd Configuration

Create the configuration directory and generate the default configuration:

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Configuration file: `/etc/containerd/config.toml`

### 8.5 Configure the Systemd Cgroup Driver

Open the configuration file:

```bash
sudo vi /etc/containerd/config.toml
```

Locate the `runc` runtime options section and set:

```toml
SystemdCgroup = true
```

Ensure the setting is under the correct runtime section for your installed containerd version. Also ensure that the CRI plugin is not disabled.

**Purpose:** Aligns containerd with the systemd cgroup driver used by the kubelet and supports cgroup v2.

### 8.6 Enable and Start containerd

```bash
sudo systemctl enable --now containerd
```

Verify the service status:

```bash
sudo systemctl status containerd
```

Expected status:

```text
Active: active (running)
```

If the service fails to start, check the logs:

```bash
sudo journalctl -u containerd -xe --no-pager
```

### 8.7 Verify the Configuration

Check whether containerd is running and enabled at boot:

```bash
sudo systemctl is-active containerd
sudo systemctl is-enabled containerd
```

Expected output:

```text
active
enabled
```

Verify the cgroup driver setting:

```bash
sudo grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

Check whether the node uses cgroup v2:

```bash
stat -fc %T /sys/fs/cgroup/
```

Expected output:

```text
cgroup2fs
```

### 8.8 Repeat on All Nodes

Complete the installation and configuration on `k8s-master`, `k8s-worker01`, and `k8s-worker02`.

---
