# ArkCase on Kubernetes — Complete RHEL Guide

> **Who this is for:** Anyone deploying ArkCase via Helm on RHEL 8/9, Rocky Linux, or AlmaLinux.  
> **What's fixed vs the original README:** Every RHEL-specific gotcha is called out, commands are verified working, and the original's incorrect assumptions about Podman/Kubernetes are corrected.

---

## Table of Contents

1. [Before You Start — Read This](#1-before-you-start--read-this)
2. [System Requirements](#2-system-requirements)
3. [Choose Your Path](#3-choose-your-path)
4. [Path A — Minikube (Local Dev/Testing)](#4-path-a--minikube-local-devtesting)
5. [Path B — kubeadm Production Cluster](#5-path-b--kubeadm-production-cluster)
6. [Install CLI Tools (kubectl + Helm)](#6-install-cli-tools-kubectl--helm)
7. [Add the ArkCase Helm Repo](#7-add-the-arkcase-helm-repo)
8. [Deploy ArkCase](#8-deploy-arkcase)
9. [Read Admin Credentials](#9-read-admin-credentials)
10. [Access the App](#10-access-the-app)
11. [Troubleshooting](#11-troubleshooting)
12. [Clean Uninstall & Reinstall](#12-clean-uninstall--reinstall)
13. [Quick Reference Cheatsheet](#13-quick-reference-cheatsheet)

---

## 1. Before You Start — Read This

### Critical Misconception in the Original README

The original README says to use **Podman as a Kubernetes runtime**. This is wrong and will not work.

Here is why:

- **Podman** is a container management tool. It does not expose the CRI (Container Runtime Interface) socket that Kubernetes (`kubelet`) requires.
- **Minikube with the Podman driver** works for local development because Minikube runs Kubernetes *inside* a Podman-managed container — Podman is the *driver*, not the runtime.
- **For a real cluster** (multi-node, production-like), you need CRI-O or containerd as the actual runtime, managed by `kubeadm`.

### Quick Decision Tree

```
Are you just testing ArkCase on a single machine?
  └─ YES → Use Path A (Minikube). Simplest setup, no production use.
  └─ NO  → Use Path B (kubeadm + CRI-O). Proper cluster, supports multi-node.
```

---

## 2. System Requirements

### Minimum Hardware

| Component | Dev (Minikube) | Production (kubeadm) |
|-----------|---------------|----------------------|
| CPU cores | 6 | 4 per node (8+ recommended) |
| RAM | 12 GB | 16 GB per node |
| Disk | 50 GB free | 100 GB per node |
| OS | RHEL 8/9, Rocky 8/9, AlmaLinux 8/9 | Same |

> **Why so much RAM?** ArkCase is a full enterprise stack: it includes a database, Solr search, Alfresco content management, Pentaho reporting, and the core application. Each component has its own pod. Under-resourcing is the #1 cause of pods stuck in `Pending`.

### Software Prerequisites

- RHEL 8, RHEL 9, Rocky Linux 8/9, or AlmaLinux 8/9
- A user with `sudo` privileges (or root)
- Internet access (or a pre-seeded private registry)

---

## 3. Choose Your Path

| Goal | Use |
|------|-----|
| Local development / testing on one machine | [Path A — Minikube](#4-path-a--minikube-local-devtesting) |
| Production or multi-node cluster | [Path B — kubeadm](#5-path-b--kubeadm-production-cluster) |

Both paths converge at [Section 6](#6-install-cli-tools-kubectl--helm) for `kubectl` and `helm`.

---

## 4. Path A — Minikube (Local Dev/Testing)

Minikube creates a single-node Kubernetes cluster on your machine. On RHEL, the recommended driver is Podman (rootful mode) with CRI-O as the internal container runtime.

> **Known issue with rootless Podman + Minikube:** DNS resolution inside the cluster often breaks when using rootless Podman. Always run Minikube as a regular user with `sudo`-capable Podman (rootful mode). If you must use rootless, you will likely hit CoreDNS timeouts and ingress failures.

### Step 4.1 — Install Podman

```bash
sudo dnf -y update
sudo dnf -y install podman
```

Verify:

```bash
podman --version
# Expected: podman version 4.x.x
```

### Step 4.2 — Configure Podman for Rootful Minikube

Minikube needs to call Podman with `sudo`. Set up passwordless sudo for Podman:

```bash
# Check your username
whoami

# Edit sudoers safely
sudo visudo
```

Add this line at the bottom (replace `yourusername` with your actual username):

```
yourusername ALL=(ALL) NOPASSWD: /usr/bin/podman
```

Save and exit (`Esc`, then `:wq` in vi).

### Step 4.3 — Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

Verify:

```bash
minikube version
```

### Step 4.4 — Install kubectl and Helm

Jump to [Section 6](#6-install-cli-tools-kubectl--helm) and come back here.

### Step 4.5 — Start Minikube

```bash
minikube start \
  --driver=podman \
  --container-runtime=cri-o \
  --cpus=6 \
  --memory=12288
```

If the above fails with a `cri-o` error (CRI-O not found inside Minikube), try without specifying the runtime — Minikube will pick a bundled one:

```bash
minikube start \
  --driver=podman \
  --cpus=6 \
  --memory=12288
```

Check the cluster is up:

```bash
kubectl get nodes
# Expected:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.x.x
```

### Step 4.6 — Enable Ingress Addon

```bash
minikube addons enable ingress

# Wait for the ingress pod to become ready
kubectl get pods -n ingress-nginx -w
# Press Ctrl+C when all pods show Running/Ready
```

> **Do not use** `helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx` alongside a Minikube ingress addon. The original README mixes both approaches, which causes conflicts. Use the addon for Minikube and skip the Helm-based ingress install.

Now skip to [Section 7](#7-add-the-arkcase-helm-repo).

---

## 5. Path B — kubeadm Production Cluster

This sets up a proper Kubernetes cluster using CRI-O as the container runtime and `kubeadm` to bootstrap. Run these steps on **every node** (control plane and workers) unless noted otherwise.

### Step 5.1 — System Prep (All Nodes)

#### a) Update the system

```bash
sudo dnf -y update
sudo dnf -y install curl wget tar
```

#### b) Disable swap

Kubernetes requires swap to be off. This is permanent:

```bash
# Turn off swap immediately
sudo swapoff -a

# Comment out the swap line in fstab to survive reboots
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab

# Verify
swapon --show
# (should show nothing)
```

#### c) Set SELinux to Permissive

SELinux in enforcing mode blocks several container networking operations. The official Kubernetes docs require it to be set to permissive. This is a runtime change + a permanent config change:

```bash
# Immediately (no reboot needed)
sudo setenforce 0

# Permanently
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Verify
getenforce
# Expected: Permissive
```

> **Security note:** Setting SELinux to permissive does not disable audit logging — violations are still recorded. This is the same approach used by OpenShift. If your security policy forbids permissive mode, you will need to write custom SELinux policies for Kubernetes, which is out of scope here.

#### d) Load required kernel modules

```bash
# Load modules now
sudo modprobe overlay
sudo modprobe br_netfilter

# Make them load automatically on boot
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

#### e) Configure kernel networking parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply immediately
sudo sysctl --system
```

#### f) Configure firewall

If `firewalld` is running, open the required ports. On a **control plane node**:

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp       # Kubernetes API server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp      # kubelet API
sudo firewall-cmd --permanent --add-port=10251/tcp      # kube-scheduler
sudo firewall-cmd --permanent --add-port=10252/tcp      # kube-controller-manager
sudo firewall-cmd --reload
```

On **worker nodes**:

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp      # kubelet API
sudo firewall-cmd --permanent --add-port=30000-32767/tcp # NodePort services
sudo firewall-cmd --reload
```

### Step 5.2 — Install CRI-O

CRI-O is Red Hat's preferred container runtime for Kubernetes on RHEL. Set the version variable to match the Kubernetes version you plan to install:

```bash
# Set this to match your intended Kubernetes version
KUBERNETES_VERSION=v1.30
CRIO_VERSION=v1.30

# Add CRI-O repository
cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF

# Install CRI-O
sudo dnf -y install cri-o

# Enable and start CRI-O
sudo systemctl enable --now crio

# Verify CRI-O is running
sudo systemctl status crio
```

### Step 5.3 — Install Kubernetes Components

```bash
# Set Kubernetes version (match CRIO_VERSION above)
KUBERNETES_VERSION=v1.30

# Add Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install Kubernetes components
sudo dnf makecache
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable kubelet (it won't start fully until after kubeadm init, that is normal)
sudo systemctl enable --now kubelet
```

### Step 5.4 — Initialize the Cluster (Control Plane Node Only)

Run this only on the control plane (master) node:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# This will print a `kubeadm join` command at the end. 
# COPY IT — you'll need it for worker nodes.
```

Set up `kubectl` access for your regular user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
# Control plane will show NotReady — that is normal until CNI is installed
```

### Step 5.5 — Install a CNI Network Plugin (Control Plane Only)

Kubernetes needs a pod network. Calico is a reliable choice for RHEL:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Wait for nodes to become Ready (may take 1-2 minutes)
kubectl get nodes -w
# Press Ctrl+C when status shows Ready
```

### Step 5.6 — Join Worker Nodes (Worker Nodes Only)

On each worker node, run the `kubeadm join` command that was printed during `kubeadm init`. It looks like this:

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Back on the control plane, verify all nodes joined:

```bash
kubectl get nodes
# All nodes should eventually show Ready
```

### Step 5.7 — Install Ingress Controller (Control Plane Only)

For a kubeadm cluster, install ingress-nginx via Helm (not Minikube addon):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Wait for ingress controller pod
kubectl get pods -n ingress-nginx -w
```

---

## 6. Install CLI Tools (kubectl + Helm)

> Skip kubectl if you already installed it in Section 5.3. Run these steps on the machine where you will run `helm` and `kubectl` commands (typically the control plane or your workstation).

### Install kubectl

#### Option A — Via Kubernetes yum repo (recommended for RHEL)

```bash
KUBERNETES_VERSION=v1.30

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubectl
```

#### Option B — Binary download

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

Verify:

```bash
kubectl version --client
```

---

### Install Helm

The original README's Helm install has a common RHEL bug: `chmod +x helm /usr/local/bin/` changes permissions on the *directory*, not the binary. The correct sequence is:

```bash
# Download the latest Helm release
HELM_VERSION=v3.17.3
curl -LO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"

# Extract
tar -zxvf "helm-${HELM_VERSION}-linux-amd64.tar.gz"

# Install the binary (correct way)
sudo install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm

# Clean up
rm -rf linux-amd64 "helm-${HELM_VERSION}-linux-amd64.tar.gz"
```

> **If you get `bash: helm: command not found` after installation**, your PATH may not include `/usr/local/bin`. Fix it:
> ```bash
> echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
> source ~/.bashrc
> ```

Verify:

```bash
helm version
# Expected: version.BuildInfo{Version:"v3.17.3", ...}
```

---

## 7. Add the ArkCase Helm Repo

Run these on any machine with `helm` and `kubectl` configured to talk to your cluster:

```bash
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update

# Verify the chart is available
helm search repo arkcase/app
```

You should see output similar to:

```
NAME           CHART VERSION   APP VERSION   DESCRIPTION
arkcase/app    x.x.x           x.x.x         ArkCase Application
```

If the search returns nothing, your Helm repo cache may be stale — run `helm repo update` again.

### Optional: Generate chart defaults for reference

```bash
mkdir -p chart-defaults
helm show values arkcase/app > chart-defaults/app-values.yaml
```

This is useful to see every configurable option before deploying.

---

## 8. Deploy ArkCase

### Step 8.1 — Create the Namespace

```bash
kubectl create namespace arkcase --dry-run=client -o yaml | kubectl apply -f -
```

The `--dry-run=client -o yaml | kubectl apply` pattern is idempotent — it is safe to run multiple times and will not error if the namespace already exists.

### Step 8.2 — Install ArkCase

From the root of this repository:

```bash
helm upgrade --install arkcase arkcase/app \
  -n arkcase \
  --create-namespace \
  -f values/dev-values.yaml \
  --timeout 20m
```

> **Why `--timeout 20m`?** The original README has no timeout. ArkCase pulls many images and initializes several services. The default Helm timeout (5 minutes) is often not enough, causing a misleading "timeout" failure even though the pods eventually come up. 20 minutes is a safe buffer.

### Step 8.3 — Watch Rollout

```bash
kubectl get pods -n arkcase -w
```

Press `Ctrl+C` when all pods show `Running` and `1/1` in the READY column. This can take 10–20 minutes on the first install as images are pulled.

#### Pods stuck in `Pending`?

This almost always means **insufficient resources**. Check:

```bash
kubectl describe pod <stuck-pod-name> -n arkcase
# Look for "Insufficient memory" or "Insufficient cpu" in the Events section
```

If you see resource errors, either increase your cluster resources or enable dev mode to reduce resource requests:

```bash
# In your values/dev-values.yaml, add or verify:
global:
  dev:
    resources: true   # This is the default — uses lower resource requests
```

---

## 9. Read Admin Credentials

ArkCase auto-generates admin credentials and stores them in a Kubernetes Secret.

### Read credentials (Bash)

```bash
echo "Username:"
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.username}' | base64 -d; echo

echo "Password:"
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### Read credentials (PowerShell — Windows)

```powershell
$u = kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.username}'
$p = kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($u))
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

### Important: Login username format

The username stored in the secret (e.g., `arkcase-admin`) must be entered with the domain suffix when logging in through the UI:

```
arkcase-admin@dev.arkcase.com
```

Using just `arkcase-admin` will return a `?login_error` even with the correct password.

### Optional: Set a custom admin password

Only do this if you want to replace the auto-generated password:

```bash
kubectl patch secret arkcase-core-main-admin -n arkcase \
  --type merge \
  -p '{
    "stringData": {
      "username": "admin",
      "password": "YourStrongPasswordHere!"
    }
  }'

# Restart the core pod to pick up the new secret
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

---

## 10. Access the App

ArkCase is configured to run at:

```
https://server.dev.arkcase.com/arkcase
```

Your machine needs to resolve this hostname to the ingress controller.

### Step 10.1 — Find Your Ingress IP

**Minikube:**

```bash
minikube ip
# Returns something like: 192.168.49.2
```

**kubeadm:**

```bash
kubectl get svc -n ingress-nginx
# Look for the EXTERNAL-IP of ingress-nginx-controller
# On bare metal this may show <pending> — see note below
```

> **kubeadm bare-metal note:** If `EXTERNAL-IP` shows `<pending>`, your cluster has no LoadBalancer provisioner. Use port-forwarding instead (see Step 10.3).

### Step 10.2 — Add a Hosts File Entry

**Linux** (`/etc/hosts`):

```bash
# Replace 192.168.49.2 with your actual ingress IP
echo "192.168.49.2  server.dev.arkcase.com" | sudo tee -a /etc/hosts
```

**Windows** (`C:\Windows\System32\drivers\etc\hosts`, run as Administrator):

```
192.168.49.2  server.dev.arkcase.com
```

### Step 10.3 — Port-Forward (If Ingress IP Is Unreachable)

If you cannot route to the ingress IP directly (common on laptops, VMs without bridged networking):

```bash
kubectl port-forward svc/ingress-nginx-controller 8444:443 -n ingress-nginx
```

Then access ArkCase at:

```
https://server.dev.arkcase.com:8444/arkcase/
```

And add this to your hosts file instead:

```
127.0.0.1  server.dev.arkcase.com
```

### Step 10.4 — Verify the Endpoint

```bash
# Should return HTTP 302 (redirect to login)
curl -k -I --resolve server.dev.arkcase.com:443:127.0.0.1 \
  https://server.dev.arkcase.com/arkcase/

# Should return HTTP 200
curl -k -I --resolve server.dev.arkcase.com:443:127.0.0.1 \
  https://server.dev.arkcase.com/arkcase/login
```

If using port-forward on 8444:

```bash
curl -k -I --resolve server.dev.arkcase.com:8444:127.0.0.1 \
  https://server.dev.arkcase.com:8444/arkcase/login
```

### Step 10.5 — Log In

Open a browser (use Incognito/Private mode to avoid cached session issues) and go to:

```
https://server.dev.arkcase.com/arkcase/
```

Login with:
- **Username:** `arkcase-admin@dev.arkcase.com` (the decoded username + `@dev.arkcase.com`)
- **Password:** the decoded password from Section 9

---

## 11. Troubleshooting

### Problem: `helm: command not found` after installation

**Cause:** `/usr/local/bin` is not in PATH, or the binary was given execute permission on the directory instead of the file.

**Fix:**

```bash
# Check if helm exists at the right path
ls -la /usr/local/bin/helm

# If it's there but not found, fix PATH
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

---

### Problem: Pods stuck in `Pending`

**Check:**

```bash
kubectl describe pod <pod-name> -n arkcase
# Look at the Events section at the bottom
```

**Common causes and fixes:**

| Event message | Cause | Fix |
|---|---|---|
| `Insufficient memory` | Not enough RAM | Add memory to nodes or reduce resource requests |
| `Insufficient cpu` | Not enough CPU | Add CPU to nodes |
| `no nodes are available that match all of the following predicates` | Resource + taint combo | Remove control-plane taint: `kubectl taint nodes --all node-role.kubernetes.io/control-plane-` |
| `PersistentVolumeClaim is not bound` | No storage provisioner | Ensure default StorageClass exists: `kubectl get storageclass` |

---

### Problem: Pods stuck in `ImagePullBackOff`

**Check:**

```bash
kubectl describe pod <pod-name> -n arkcase
# Look for "Failed to pull image" in Events
```

**Common cause:** Rate limiting on Docker Hub or network access issues.

**Fix:**

```bash
# Test connectivity from a node
curl -I https://registry-1.docker.io/
```

If behind a corporate proxy, set `HTTP_PROXY` and `HTTPS_PROXY` for the container runtime. For CRI-O:

```bash
sudo mkdir -p /etc/systemd/system/crio.service.d/
cat <<EOF | sudo tee /etc/systemd/system/crio.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
EOF
sudo systemctl daemon-reload
sudo systemctl restart crio
```

---

### Problem: Login fails with correct credentials

**Cause A:** Domain suffix missing from username.

**Fix:** Use `arkcase-admin@dev.arkcase.com`, not `arkcase-admin`.

**Cause B:** The `arkcase-core` pod cached the old credentials.

**Fix:**

```bash
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

Then try again in a fresh Incognito/Private browser window.

**Cause C:** Browser has a cached session from a previous install.

**Fix:** Clear cookies for `server.dev.arkcase.com`, or use a different browser/Incognito window.

---

### Problem: `kubectl port-forward` fails — address already in use

**Fix:**

```bash
# Check what's using the port (Linux)
sudo ss -tlnp | grep 8443

# Kill that process, or use an alternate local port:
kubectl port-forward -n arkcase svc/core 9443:8443
```

---

### Problem: `kubectl get pods -l app.kubernetes.io/name=arkcase` shows no results

**Cause:** ArkCase pods use a different label selector.

**Fix:**

```bash
# Correct label selectors for ArkCase
kubectl get pods -n arkcase -l app.kubernetes.io/instance=arkcase

# Or just list everything in the namespace
kubectl get pods -n arkcase
```

---

### Problem: Helm install fails with `another operation is in progress`

**Cause:** A previous Helm operation was interrupted and left the release in a bad state.

**Fix:**

```bash
# Check the release status
helm list -A

# If status shows pending-install or pending-upgrade, force-delete the stuck release
helm delete arkcase -n arkcase --no-hooks

# Then reinstall
helm upgrade --install arkcase arkcase/app -n arkcase -f values/dev-values.yaml
```

---

### Problem: CRI-O fails to start on RHEL 8

**Check:**

```bash
sudo journalctl -u crio -n 50
```

A common error is a missing or misconfigured runtime:

```bash
# Ensure conmon and runc/crun are installed
sudo dnf install -y cri-o conmon runc
sudo systemctl restart crio
```

---

## 12. Clean Uninstall & Reinstall

### Uninstall ArkCase

```bash
helm uninstall arkcase -n arkcase

# Verify removal
helm list -A
kubectl get pods -n arkcase
```

### (Optional) Delete persistent data

> ⚠️ This is destructive and permanent. All ArkCase data will be lost.

```bash
kubectl get pvc -n arkcase
kubectl delete pvc --all -n arkcase
```

### Reinstall from scratch

```bash
helm upgrade --install arkcase arkcase/app \
  -n arkcase \
  --create-namespace \
  -f values/dev-values.yaml \
  --timeout 20m

# Watch pods come up
kubectl get pods -n arkcase -w
```

---

## 13. Quick Reference Cheatsheet

```bash
# ── CLUSTER STATUS ────────────────────────────────────────────────
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n arkcase
kubectl get pods -n arkcase -l app.kubernetes.io/instance=arkcase

# ── HELM ──────────────────────────────────────────────────────────
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update
helm search repo arkcase/app
helm list -A
helm upgrade --install arkcase arkcase/app -n arkcase \
  --create-namespace -f values/dev-values.yaml --timeout 20m
helm uninstall arkcase -n arkcase

# ── CREDENTIALS ───────────────────────────────────────────────────
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.password}' | base64 -d; echo

# ── ACCESS ────────────────────────────────────────────────────────
# Minikube ingress IP
minikube ip

# Port-forward ingress (when IP not routable)
kubectl port-forward svc/ingress-nginx-controller 8444:443 -n ingress-nginx

# Port-forward directly to core
kubectl port-forward svc/core 9443:8443 -n arkcase

# ── INGRESS ───────────────────────────────────────────────────────
kubectl get ingress -n arkcase
kubectl describe ingress -n arkcase

# ── DEBUGGING ─────────────────────────────────────────────────────
kubectl describe pod <pod-name> -n arkcase
kubectl logs <pod-name> -n arkcase
kubectl logs <pod-name> -n arkcase --previous  # logs from last crash

# Restart the core statefulset
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s

# ── CLEANUP ───────────────────────────────────────────────────────
kubectl get pvc -n arkcase
kubectl delete pvc --all -n arkcase   # WARNING: destroys all data
```

---

## Appendix: values/dev-values.yaml Key Settings

The `values/dev-values.yaml` file in this repo overrides the chart defaults for local development. Key settings:

```yaml
global:
  settings:
    baseUrl: "https://server.dev.arkcase.com/arkcase"
  ingress:
    className: "nginx"
  dev:
    resources: true   # Enables lower resource requests for dev
```

The ingress annotations ensure ArkCase's HTTPS backend is handled correctly:

```yaml
# These annotations tell nginx to proxy to an HTTPS backend
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

To see all configurable values:

```bash
helm show values arkcase/app | less
```

---

*Guide last updated: March 2026. Tested against RHEL 8/9, Rocky Linux 8/9, Minikube v1.33+, Kubernetes v1.30, Helm v3.17, ArkCase chart series 25.x.*
