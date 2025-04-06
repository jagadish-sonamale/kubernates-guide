# ⚠️ Disclaimer

> This guide is intended for learning, development, and testing purposes only. For production-grade setups, always refer to the official Kubernetes documentation: [https://kubernetes.io/docs/setup/](https://kubernetes.io/docs/setup/).

---

# 🌍 Kubernetes Overview

Kubernetes is an open-source platform that automates the deployment, scaling, and management of containerized applications. It groups containers into logical units called Pods, and distributes them across a cluster of nodes to ensure high availability and fault tolerance.

### ✨ Key Benefits:
- Automates application deployment and scaling
- Handles container orchestration
- Self-healing (reschedules failed pods automatically)
- Supports declarative configuration and desired state management

---

## 🏗️ Kubernetes Architecture

Kubernetes follows a client-server architecture consisting of:

### Control Plane (Master Components):
- **kube-apiserver**: Frontend API that all kubectl and internal requests hit.
- **etcd**: Stores the entire state/configuration of the cluster.
- **kube-scheduler**: Assigns pods to nodes.
- **kube-controller-manager**: Maintains desired state by running controllers (e.g., node, replication).

### Node Components (Workers):
- **kubelet**: Communicates with the control plane to manage pods and containers.
- **container runtime (e.g., containerd)**: Runs the containers.
- **kube-proxy**: Manages networking rules for pod-to-pod and pod-to-service traffic.

### Add-ons:
- **CNI Plugin (e.g., Calico)**: Enables networking between pods.
- **DNS**: Adds DNS-based service discovery inside the cluster.

---

# ☸️ Kubernetes Setup Notes – Explained (Single Node using `kubeadm` on Ubuntu)


## 📝 Summary

This guide provides a complete and beginner-friendly walkthrough to set up a **single-node Kubernetes cluster** using `kubeadm` on Ubuntu. It covers:

- System preparation (swap disable, IP forwarding, updates)
- Installing and configuring the container runtime (`containerd`)
- Installing Kubernetes components (`kubeadm`, `kubelet`, `kubectl`)
- Initializing the Kubernetes control plane
- Installing a network plugin (Calico)
- Allowing scheduling of pods on the control-plane node
- Validating the cluster setup
- Common kubectl commands
- Detailed clean-up instructions to reset or remove the cluster completely

This setup is ideal for learning and development environments.

---

## ✅ 1. Prerequisites

- A fresh Ubuntu machine (e.g., EC2 instance)
- Minimum 2 CPUs, 2 GB RAM (K8s components are resource-intensive)
- `sudo` privileges
- Outbound internet connectivity (required to pull images and apply network manifests)

---

## 🔧 2. System Preparation

### 🧱 Step 1: Update the system

```bash
sudo apt update && sudo apt upgrade -y
```
- `sudo apt update`: Refreshes the local package index by retrieving the latest list of available packages from repositories. This ensures that the system knows about the most recent versions.
- `sudo apt upgrade -y`: Installs newer versions of the packages that are already installed on the system. The `-y` flag auto-confirms prompts.
- 🔍 **Why is this important?**
  - Ensures all dependencies and base libraries are up-to-date and compatible with the Kubernetes components we are about to install.
  - Helps avoid unexpected behavior caused by outdated or broken packages.
  - Some Kubernetes tools may rely on updated system packages, libraries, or kernel modules to function properly.
  - Boosts system security by applying latest patches before provisioning a production-grade or dev-grade environment.
- `apt update`: Refreshes the local package index.
- `apt upgrade -y`: Automatically installs the latest versions of available packages.
- Keeping your system updated ensures security patches and newer software compatibility.

### 🧱 Step 2: Disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
- `swapoff -a`: Turns off swap memory temporarily. It stops the system from using swap space right now.
- `sed -i '/ swap / s/^/#/' /etc/fstab`: Comments out the swap entry in the system’s startup file so that swap remains disabled after reboot.

🔍 **Why is this important?**
- Kubernetes expects to manage memory itself for all running containers. If swap is enabled, the operating system might move parts of memory to disk, which can confuse Kubernetes.
- Having swap on can lead to unstable pod performance. Some pods might be forcefully killed when memory is tight.
- kubeadm will stop and show an error if swap is detected. So this step is mandatory.
- By disabling swap, we ensure that Kubernetes has full control over memory management for better consistency and performance.
- Kubernetes components (like `kubelet`) are designed to manage memory resources themselves. Swap can interfere with Kubernetes scheduling and memory constraints.
- With swap enabled, nodes might not behave consistently under memory pressure, and this can cause pods to be OOM-killed (Out of Memory) or experience unexpected latency.
- The `kubeadm` init process will fail with a pre-flight error if swap is detected. Disabling it is required to proceed.
- Disabling swap ensures a more predictable and stable Kubernetes runtime environment, especially for memory management across pods and containers.
- `sed` command comments out the swap line in `/etc/fstab` to prevent it from being re-enabled after a reboot.
- Kubernetes requires swap to be disabled for optimal memory management and performance.

### 🧱 Step 3: Enable IP forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```
- Temporarily enables IP forwarding.

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
- Appends this setting to `/etc/sysctl.conf` to make it permanent and reloads the sysctl settings.
- IP forwarding is essential for network communication between pods across nodes.

---

## 📦 3. Install and Configure containerd (Container Runtime)

### 🧱 Step 1: Install containerd

```bash
sudo apt install -y containerd
```
- Installs containerd, a lightweight and efficient container runtime supported by Kubernetes.

### 🧱 Step 2: Generate default config

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
- Creates the configuration directory and writes the default configuration to `/etc/containerd/config.toml`.

### 🧱 Step 3: Set `SystemdCgroup = true`

Edit the config:
```bash
sudo nano /etc/containerd/config.toml
```

Find and change:
```toml
SystemdCgroup = true
```
- Aligns containerd’s cgroup driver with Kubernetes which uses `systemd`.

### 🧱 Step 4: Restart and enable containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```
- Restarts containerd to apply changes.
- Enables it to start automatically on boot.

---

## ☸️ 4. Install Kubernetes Binaries (`kubeadm`, `kubelet`, `kubectl`)

### 🧱 Step 1: Add Kubernetes repository and GPG key

```bash
sudo apt install -y apt-transport-https ca-certificates curl
```
- Ensures your system can download packages over HTTPS.

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
- Downloads and stores Google’s GPG key to verify package authenticity.

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
https://apt.kubernetes.io/ kubernetes-xenial main" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```
- Adds Kubernetes' official package source to your system.

### 🧱 Step 2: Install Kubernetes components

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
- `kubelet`: Agent that runs on each node.
- `kubeadm`: Tool to bootstrap the cluster.
- `kubectl`: CLI tool to interact with the cluster.
- `apt-mark hold`: Prevents automatic updates that might break compatibility.

---

## 🚀 5. Initialize the Kubernetes Cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
- Starts the cluster with a specific pod network range compatible with most CNIs like Calico.
- Prints a join command used for adding worker nodes later.

---

## 🧐 6. Configure kubectl (Cluster Access)

```bash
mkdir -p $HOME/.kube
```
- Creates the `.kube` directory for your user.

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
- Copies the admin configuration file to your user's kubeconfig path.

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Grants you permission to use the `kubectl` config.

---

## 🌐 7. Install Pod Network Plugin (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
- Deploys Calico, a network plugin that manages pod-to-pod communication.
- It also provides NetworkPolicy features for fine-grained traffic control.

---

## 🧱 8. Allow Control Plane to Schedule Pods (Single Node)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
- Removes the default taint on control-plane node that prevents pod scheduling.
- Needed for single-node clusters so all pods can run on the same node.

---

## ✅ 9. Validate Cluster Status

```bash
kubectl get nodes
```
- Verifies node is in `Ready` status.

```bash
kubectl get pods -A
```
- Lists all pods in all namespaces to confirm everything is running correctly.

---

## 🛠️ 10. Common kubectl Commands

```bash
kubectl get pods -A
```
- Shows all pods in all namespaces.

```bash
kubectl apply -f <filename>.yaml
```
- Creates or updates resources defined in a YAML file.

```bash
kubectl delete -f <filename>.yaml
```
- Deletes resources defined in the given YAML file.

```bash
kubectl describe pod <pod-name> -n <namespace>
```
- Provides detailed information about a specific pod.

```bash
kubectl logs <pod-name>
```
- Retrieves logs from the specified pod.

---

## 💚 11. Clean-up (Teardown)

### 🔧 Reset the cluster:
```bash
sudo kubeadm reset -f
```
- Removes the Kubernetes control plane components and cleans cluster state.

### 🔧 Remove configuration files:
```bash
sudo rm -rf /etc/cni /etc/kubernetes /var/lib/etcd /var/lib/kubelet ~/.kube
```
- Deletes config files, CNI setup, kubelet state, and user configs.

### 🔧 Remove container networking interfaces:
```bash
sudo ip link delete cni0
sudo ip link delete flannel.1
```
- Cleans up virtual network interfaces (depends on the CNI used).

### 🔧 Remove containerd configs (optional):
```bash
sudo rm -rf /etc/containerd/config.toml
```
- Useful if you're planning to reconfigure containerd or start fresh.

---

> With this, you have a clean slate to re-init or remove the cluster completely.

---

This document gives you a complete picture of setting up and tearing down a single-node Kubernetes cluster. Always refer to the official site for updated commands and version-specific instructions.

---

## 📘 Kubernetes Core Components Explained

### 1. `kubeadm`
- A command-line tool used to bootstrap a Kubernetes cluster.
- Initializes the control-plane (API server, etcd, controller manager, scheduler).
- Generates certificates, kubeconfig files, and handles cluster token creation.

### 2. `kubelet`
- An agent that runs on every node in the cluster.
- Communicates with the API server and ensures containers described in PodSpecs are running.
- Watches for Pod definitions and reports the status back to the control plane.

### 3. `kubectl`
- A command-line interface tool to interact with the Kubernetes cluster.
- Sends API requests to the control-plane (via the API server).
- Allows users to deploy apps, inspect logs, check cluster health, and perform other management tasks.

### 4. `containerd`
- A container runtime that runs and manages containers on the node.
- Responsible for pulling images, starting containers, and managing container lifecycle.
- Kubernetes communicates with containerd through the Container Runtime Interface (CRI).

### 5. `etcd`
- A distributed key-value store used as Kubernetes' backing store.
- Stores the entire state of the Kubernetes cluster: nodes, pods, secrets, config maps, etc.
- Highly available and consistent.

### 6. `kube-apiserver`
- Frontend of the Kubernetes control plane.
- All operations (kubectl commands, internal component requests) go through the API server.
- Validates and processes REST requests, then updates etcd and notifies other components.

### 7. `kube-controller-manager`
- Runs multiple controllers in a single process.
- Each controller watches for specific resources in the API server and takes action to reconcile actual state with desired state.
  - Node controller, replication controller, endpoints controller, etc.

### 8. `kube-scheduler`
- Watches for new Pods without a node assignment.
- Selects the best node for a Pod to run on based on resource requirements, affinity, taints, etc.

### 9. Network Plugin (CNI like Calico)
- Handles pod-to-pod communication across nodes.
- Sets up virtual interfaces and assigns IPs to pods.
- Implements network policy for controlling ingress/egress traffic.

These components work together to form a functioning Kubernetes cluster. Understanding them helps in managing, troubleshooting, and scaling clusters effectively.

---

## 🧩 Alternative Options and Comparisons

### Container Runtimes
| Runtime       | Description                                                                 | Recommended For                       |
|---------------|-----------------------------------------------------------------------------|----------------------------------------|
| containerd     | Lightweight, fast, and CRI-compatible. Default in many modern setups.      | Most general-purpose Kubernetes clusters |
| CRI-O          | Designed specifically for Kubernetes and OpenShift.                         | OpenShift or minimal K8s setups        |
| Docker (deprecated for K8s) | Previously default but now deprecated as Kubernetes runtime. | Legacy clusters (use with caution)     |

### Network Plugins (CNI)
| Plugin     | Description                                                   | Pros                                      | Use Case                        |
|------------|---------------------------------------------------------------|-------------------------------------------|----------------------------------|
| Calico     | Policy engine + CNI plugin. Supports BGP, IPIP, VXLAN.       | Highly customizable; supports NetworkPolicy | Secure and scalable networks    |
| Flannel    | Simple overlay network using VXLAN.                          | Easy to set up; low overhead               | Lightweight dev/test clusters   |
| Weave Net  | Mesh-based overlay with encryption support.                  | Simpler to understand; secure by default   | Small-scale secure clusters     |
| Cilium     | eBPF-based high performance network plugin.                  | High performance; rich observability       | Performance-critical applications |

### Cluster Bootstrap Tools
| Tool       | Description                                                        | Use Case                            |
|------------|--------------------------------------------------------------------|--------------------------------------|
| kubeadm    | Official tool to set up and manage cluster components manually.   | Dev, testing, custom production     |
| k3s        | Lightweight Kubernetes distribution for edge/IoT.                | Edge, low-resource devices          |
| kind       | Runs Kubernetes in Docker containers.                            | Local dev/testing environments      |
| Minikube   | Runs a single-node cluster in a VM or Docker.                    | Local dev, beginners                 |

These options allow flexibility based on environment, scale, and goals. Choosing the right combination ensures efficiency and long-term maintainability of your Kubernetes setup.

---

## 🧠 What Else You Should Know

### ✅ Security Considerations
- Always use Role-Based Access Control (RBAC) to restrict access.
- Protect the kubeconfig file—it contains credentials to access the cluster.
- Enable audit logging and TLS for secure communication between components.

### ✅ Monitoring & Observability
- Use tools like Prometheus and Grafana for real-time metrics.
- Use `kubectl top` to monitor resource usage.
- Enable logging with tools like EFK (Elasticsearch, Fluentd, Kibana) or Loki.

### ✅ Backup & Restore
- Regularly back up `etcd` (it stores all cluster state).
- Use Velero or etcd snapshot tools for backup and disaster recovery.

### ✅ Upgrades
- Use `kubeadm upgrade` for upgrading clusters safely.
- Always back up before upgrading.

### ✅ Best Practices
- Use namespaces to organize workloads.
- Apply resource requests and limits to every Pod.
- Use liveness and readiness probes for health checks.
- Don’t run unnecessary workloads on the control-plane node (except for single-node dev clusters).

This extra knowledge helps in running Kubernetes clusters reliably in real-world environments, especially beyond basic setups.

