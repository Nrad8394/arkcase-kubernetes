# ArkCase on Kubernetes — Complete Cross-Platform Guide

> **Platforms covered:** Ubuntu / Debian · RHEL / Fedora / Rocky Linux / AlmaLinux · Windows
>
> **What this fixes vs the original README:** Incorrect Podman/Kubernetes assumptions, missing
> OS prerequisites, a broken Helm install command, no Windows production path, no Ubuntu path
> at all, and no guidance on what ArkCase actually tests against. All corrected here.

---

## Table of Contents

1. [Before You Start — Read This](#1-before-you-start--read-this)
2. [System Requirements](#2-system-requirements)
3. [Choose Your Path](#3-choose-your-path)
4. [OS Prerequisites](#4-os-prerequisites)
   - [4a. Ubuntu / Debian](#4a-ubuntu--debian)
   - [4b. RHEL / Fedora / Rocky / AlmaLinux](#4b-rhel--fedora--rocky--almalinux)
   - [4c. Windows](#4c-windows)
5. [Cluster Setup — K3s ⭐ Recommended (Linux only)](#5-cluster-setup--k3s--recommended-linux-only)
6. [Cluster Setup — Minikube (All platforms, more complex)](#6-cluster-setup--minikube-all-platforms-more-complex)
7. [Cluster Setup — kubeadm (Production multi-node, Linux only)](#7-cluster-setup--kubeadm-production-multi-node-linux-only)
8. [Cluster Setup — Docker Desktop (Windows / Mac)](#8-cluster-setup--docker-desktop-windows--mac)
9. [Install Helm](#9-install-helm)
10. [Add the ArkCase Helm Repo](#10-add-the-arkcase-helm-repo)
11. [Deploy ArkCase](#11-deploy-arkcase)
12. [Read Admin Credentials](#12-read-admin-credentials)
13. [Access the App](#13-access-the-app)
14. [Troubleshooting](#14-troubleshooting)
15. [Clean Uninstall & Reinstall](#15-clean-uninstall--reinstall)
16. [Quick Reference Cheatsheet](#16-quick-reference-cheatsheet)
17. [Appendix: values/dev-values.yaml](#17-appendix-valuesdev-valuesyaml)

---

## 1. Before You Start — Read This

### What ArkCase Actually Tests Against

ArkCase's official Helm charts are tested against **vanilla Kubernetes**, **Rancher Desktop**,
and **EKS**. Minikube, K3s, and OpenShift are described as "should work" but are not officially
validated. Every problem you hit with an untested stack is yours to debug.

**K3s is the closest thing to "vanilla Kubernetes" you can run on a single Linux machine**,
which is why it is the recommended path in this guide for Linux users.
**Docker Desktop with Kubernetes** is the recommended path for Windows.

### The Podman Misconception (Original README Was Wrong)

The original README implied Podman could act as a Kubernetes runtime. **It cannot.**

- **Podman** is a container tool. It has no CRI (Container Runtime Interface) socket, which
  is what Kubernetes (`kubelet`) requires.
- **Minikube with the Podman driver** is different — Minikube runs Kubernetes *inside* a
  Podman-managed container. Podman is the *VM driver*, not the runtime.
- On RHEL 9 with rootless Podman, Minikube hits additional bugs: broken DNS,
  `/run/runc` access errors, stale volume state. These are real documented issues.

### Quick Decision Tree

```
What OS are you on?
  ├─ Linux (Ubuntu, Debian, RHEL, Rocky, Fedora, AlmaLinux)
  │    ├─ Single machine dev/test  →  Section 5: K3s ⭐  (recommended)
  │    ├─ Already using Minikube   →  Section 6: Minikube (read caveats)
  │    └─ Production multi-node   →  Section 7: kubeadm + CRI-O
  │
  └─ Windows
       ├─ Dev/test                 →  Section 8: Docker Desktop ⭐
       └─ Already using Minikube   →  Section 6: Minikube
```

---

## 2. System Requirements

### Minimum Hardware (all platforms)

| Component | Single-node dev | Production (per node) |
|-----------|----------------|-----------------------|
| CPU cores | 6              | 4 (8+ recommended)    |
| RAM       | 12 GB          | 16 GB                 |
| Disk      | 50 GB free     | 100 GB                |

> **Why so much?** ArkCase runs an entire enterprise stack as separate pods: the core app,
> PostgreSQL/MariaDB, Solr, Alfresco, ActiveMQ, Samba LDAP, and Pentaho reporting.
> Under-resourcing is the #1 cause of pods stuck in `Pending`.

### Supported OS

| OS | Versions | Notes |
|----|----------|-------|
| Ubuntu | 20.04, 22.04, 24.04 LTS | Fully supported |
| Debian | 11, 12 | Fully supported |
| RHEL | 8, 9 | SELinux config required |
| Rocky Linux | 8, 9 | Same as RHEL |
| AlmaLinux | 8, 9 | Same as RHEL |
| Fedora | 38+ | Same as RHEL family |
| Windows | 10/11, Server 2019/2022 | Via Docker Desktop or WSL2 |

---

## 3. Choose Your Path

| Section | Platform | Best for | Complexity |
|---------|----------|----------|------------|
| [5 — K3s](#5-cluster-setup--k3s--recommended-linux-only) ⭐ | Linux | Dev/test, single machine | Low |
| [6 — Minikube](#6-cluster-setup--minikube-all-platforms-more-complex) | All | If you must use Minikube | Medium–High |
| [7 — kubeadm](#7-cluster-setup--kubeadm-production-multi-node-linux-only) | Linux | Production multi-node | High |
| [8 — Docker Desktop](#8-cluster-setup--docker-desktop-windows--mac) ⭐ | Windows / Mac | Dev/test | Low |

**All paths converge at [Section 9 — Install Helm](#9-install-helm).**

---

## 4. OS Prerequisites

Run the section matching your OS before doing anything with Kubernetes.

---

### 4a. Ubuntu / Debian

#### Update the system

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget tar ca-certificates gnupg lsb-release
```

#### Disable swap

```bash
sudo swapoff -a
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab

# Verify (should return nothing)
swapon --show
```

#### Load kernel modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

#### Set kernel networking parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

#### Firewall (if ufw is active)

```bash
sudo ufw allow 6443/tcp    # Kubernetes / K3s API
sudo ufw allow 10250/tcp   # kubelet
sudo ufw allow 443/tcp     # HTTPS ingress
sudo ufw allow 80/tcp      # HTTP ingress
```

> **Note:** Ubuntu/Debian does not use SELinux by default, so you do not need the SELinux
> permissive steps that RHEL requires.

Now continue to your chosen cluster section ([Section 5](#5-cluster-setup--k3s--recommended-linux-only)
or [Section 6](#6-cluster-setup--minikube-all-platforms-more-complex)).

---

### 4b. RHEL / Fedora / Rocky / AlmaLinux

#### Update the system

```bash
sudo dnf -y update
sudo dnf -y install curl wget tar
```

#### Disable swap

```bash
sudo swapoff -a
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab

swapon --show
# Should return nothing
```

#### Set SELinux to Permissive

SELinux enforcing mode blocks several container networking operations. Permissive mode logs
violations without blocking them — the same approach used by OpenShift:

```bash
# Apply immediately (no reboot needed)
sudo setenforce 0

# Make it permanent across reboots
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Verify
getenforce
# Expected: Permissive
```

#### Load kernel modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

#### Set kernel networking parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

#### Configure firewall (firewalld)

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp    # K3s / Kubernetes API
sudo firewall-cmd --permanent --add-port=10250/tcp   # kubelet
sudo firewall-cmd --permanent --add-port=443/tcp     # HTTPS ingress
sudo firewall-cmd --permanent --add-port=80/tcp      # HTTP ingress
sudo firewall-cmd --reload
```

Now continue to your chosen cluster section ([Section 5](#5-cluster-setup--k3s--recommended-linux-only)
or [Section 6](#6-cluster-setup--minikube-all-platforms-more-complex)).

---

### 4c. Windows

Windows does not run Kubernetes natively. Your options are:

- **Docker Desktop with Kubernetes enabled** — recommended, see [Section 8](#8-cluster-setup--docker-desktop-windows--mac)
- **Minikube inside WSL2** — possible but complex, see [Section 6](#6-cluster-setup--minikube-all-platforms-more-complex)
- **K3s inside WSL2** — works well if you are comfortable with WSL2

For most Windows users, Docker Desktop is the right choice. Skip to [Section 8](#8-cluster-setup--docker-desktop-windows--mac).

---

## 5. Cluster Setup — K3s ⭐ Recommended (Linux only)

K3s is a fully conformant, single-binary Kubernetes distribution. It bundles containerd as its
runtime, needs no VM driver, and works cleanly on Ubuntu and RHEL family systems. It is the
lowest-friction path to a working ArkCase cluster on a single Linux machine.

> **Complete [Section 4](#4-os-prerequisites) for your OS before running these steps.**

### Step 5.1 — Install K3s

```bash
# Installs K3s and disables the default Traefik ingress
# (we replace it with nginx ingress to match ArkCase's chart defaults)
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

K3s will: download its binary, configure containerd, start the Kubernetes control plane,
and register a systemd service that auto-starts on boot. Wait a few seconds, then verify:

```bash
sudo k3s kubectl get nodes
# Expected:
# NAME        STATUS   ROLES                  AGE   VERSION
# hostname    Ready    control-plane,master   30s   v1.x.x
```

### Step 5.2 — Configure kubectl Access

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

kubectl get nodes
# Same output as above, but without sudo
```

### Step 5.3 — Install kubectl (Standalone Binary)

K3s includes kubectl internally. Install the standalone binary for convenience:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

kubectl version --client
```

### Step 5.4 — Install ingress-nginx

First install Helm ([Section 9](#9-install-helm)), then come back and run:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.https=32443

# Wait for the controller pod
kubectl get pods -n ingress-nginx -w
# Ctrl+C when STATUS shows Running and READY shows 1/1
```

Now skip to [Section 9 — Install Helm](#9-install-helm).

---

## 6. Cluster Setup — Minikube (All platforms, more complex)

> ⚠️ **RHEL 9 users:** Minikube with rootless Podman has documented issues — stale volume state,
> broken DNS, `/run/runc` errors. You will likely hit at least one of these. K3s (Section 5)
> avoids all of them. Only use Minikube if you have a specific reason.
>
> **Ubuntu / Windows users:** Minikube works more reliably here, but Docker Desktop (Windows,
> Section 8) or K3s (Ubuntu, Section 5) are still simpler.

### Step 6.1 — Install a Container Driver

Minikube needs a driver to create its internal Kubernetes VM. Choose based on your platform:

**Ubuntu / Debian — Docker driver (recommended):**

```bash
# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker

# Add your user to the docker group (log out and back in after this)
sudo usermod -aG docker $USER
```

**RHEL / Rocky / AlmaLinux — Podman driver (rootful mode required):**

```bash
sudo dnf -y install podman runc

# Set up passwordless sudo for Podman (required for rootful Minikube)
sudo visudo
# Add at the bottom (replace yourusername):
# yourusername ALL=(ALL) NOPASSWD: /usr/bin/podman
```

**Windows — Hyper-V or Docker Desktop driver:**

Minikube on Windows requires either Hyper-V (enable in Windows Features) or Docker Desktop
installed first. Then install Minikube from:
`https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe`

### Step 6.2 — Install Minikube (Linux)

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install -o root -g root -m 0755 minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

minikube version
```

### Step 6.3 — Clean Any Broken State

If you have a failed previous Minikube install:

```bash
minikube delete --all --purge

# RHEL/Podman only — clean leftover Podman state
podman volume rm minikube 2>/dev/null || true
podman rm -f minikube 2>/dev/null || true
podman network rm minikube 2>/dev/null || true
minikube config unset rootless
unset MINIKUBE_ROOTLESS
```

### Step 6.4 — Start Minikube

**Ubuntu / Debian (Docker driver):**

```bash
minikube start \
  --driver=docker \
  --cpus=6 \
  --memory=12288
```

**RHEL / Rocky / AlmaLinux (Podman driver, rootful):**

```bash
unset MINIKUBE_ROOTLESS
minikube config unset rootless

minikube start \
  --driver=podman \
  --rootless=false \
  --cpus=6 \
  --memory=12288
```

**Windows (Hyper-V driver):**

```powershell
minikube start --driver=hyperv --cpus=6 --memory=12288
```

**Windows (Docker Desktop driver):**

```powershell
minikube start --driver=docker --cpus=6 --memory=12288
```

Verify:

```bash
kubectl get nodes
# Expected:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.x.x
```

### Step 6.5 — Enable Ingress Addon

```bash
minikube addons enable ingress

kubectl get pods -n ingress-nginx -w
# Ctrl+C when all pods show Running and 1/1 Ready
```

> **Do not** also install ingress-nginx via Helm when using the Minikube addon. The original
> README mixed both approaches and caused conflicts.

Now continue to [Section 9 — Install Helm](#9-install-helm).

---

## 7. Cluster Setup — kubeadm (Production multi-node, Linux only)

Use this path for a real multi-node cluster. Run **all steps on every node** unless marked
"control plane only". Works on both Ubuntu/Debian and RHEL family.

### Step 7.1 — System Prep (All Nodes)

Complete [Section 4](#4-os-prerequisites) for your OS on every node first.

**Extra firewall ports — Control Plane:**

```bash
# Ubuntu/Debian
sudo ufw allow 6443/tcp && sudo ufw allow 2379:2380/tcp && sudo ufw allow 10250:10252/tcp

# RHEL family
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250-10252/tcp
sudo firewall-cmd --reload
```

**Extra firewall ports — Worker Nodes:**

```bash
# Ubuntu/Debian
sudo ufw allow 10250/tcp && sudo ufw allow 30000:32767/tcp

# RHEL family
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

### Step 7.2 — Install containerd (All Nodes)

containerd is the standard Kubernetes runtime and works on both Ubuntu and RHEL:

**Ubuntu / Debian:**

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup driver (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```

**RHEL / Rocky / AlmaLinux:**

```bash
# containerd is distributed via Docker's repo on RHEL
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```

### Step 7.3 — Install Kubernetes Components (All Nodes)

**Ubuntu / Debian:**

```bash
KUBERNETES_VERSION=v1.30

curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

**RHEL / Rocky / AlmaLinux:**

```bash
KUBERNETES_VERSION=v1.30

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf makecache
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

### Step 7.4 — Initialize the Cluster (Control Plane Only)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> ⚠️ The output prints a `kubeadm join` command. **Copy it now** — you need it for Step 7.6.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
# Control plane shows NotReady — normal until CNI is installed
```

### Step 7.5 — Install CNI (Control Plane Only)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

kubectl get nodes -w
# Ctrl+C when all nodes show Ready
```

### Step 7.6 — Join Worker Nodes (Workers Only)

Run the `kubeadm join` command printed in Step 7.4:

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Verify from the control plane:

```bash
kubectl get nodes
```

### Step 7.7 — Install Ingress Controller (Control Plane Only)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

kubectl get pods -n ingress-nginx -w
```

---

## 8. Cluster Setup — Docker Desktop (Windows / Mac)

Docker Desktop is the recommended path for Windows and macOS. It ships with a built-in
single-node Kubernetes cluster backed by containerd — no VM driver configuration needed.

### Step 8.1 — Install Docker Desktop

Download from: `https://www.docker.com/products/docker-desktop/`

### Step 8.2 — Enable Kubernetes

1. Open Docker Desktop
2. Go to **Settings → Kubernetes**
3. Check **Enable Kubernetes**
4. Click **Apply & Restart**
5. Wait for the Kubernetes status indicator (bottom left) to turn green and show `Running`

### Step 8.3 — Verify

**PowerShell or Command Prompt:**

```powershell
kubectl cluster-info
kubectl get nodes
# Expected:
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   1m    v1.x.x
```

### Step 8.4 — Install kubectl (Windows)

Docker Desktop installs kubectl automatically. If it is not found:

```powershell
# PowerShell (run as Administrator)
winget install Kubernetes.kubectl
```

Or download the binary manually:
`https://dl.k8s.io/release/v1.30.0/bin/windows/amd64/kubectl.exe`
and place it somewhere on your `PATH`.

### Step 8.5 — Install ingress-nginx

Install Helm first ([Section 9](#9-install-helm)), then:

```powershell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx `
  --namespace ingress-nginx `
  --create-namespace

kubectl get pods -n ingress-nginx -w
# Ctrl+C when STATUS shows Running
```

Now continue to [Section 9 — Install Helm](#9-install-helm).

---

## 9. Install Helm

Run on the machine where you execute `helm` and `kubectl` commands.

### Linux (Ubuntu, Debian, RHEL, Rocky, Fedora)

```bash
HELM_VERSION=v3.17.3
curl -LO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"
tar -zxvf "helm-${HELM_VERSION}-linux-amd64.tar.gz"

# Correct install — sets permissions on the binary, not the directory
sudo install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm

rm -rf linux-amd64 "helm-${HELM_VERSION}-linux-amd64.tar.gz"

helm version
```

> **`helm: command not found` after install?**
> ```bash
> echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
> source ~/.bashrc
> ```

### Windows (PowerShell)

```powershell
# Option A — winget (simplest)
winget install Helm.Helm

# Option B — Chocolatey
choco install kubernetes-helm

# Option C — manual binary
$version = "v3.17.3"
Invoke-WebRequest "https://get.helm.sh/helm-$version-windows-amd64.zip" -OutFile helm.zip
Expand-Archive helm.zip -DestinationPath helm-extract
Move-Item helm-extract\windows-amd64\helm.exe C:\Windows\System32\helm.exe
Remove-Item -Recurse helm.zip, helm-extract
```

Verify:

```powershell
helm version
```

---

## 10. Add the ArkCase Helm Repo

Run on any machine with `helm` and `kubectl` configured:

```bash
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update

# Verify
helm search repo arkcase/app
```

Expected output:

```
NAME           CHART VERSION   APP VERSION   DESCRIPTION
arkcase/app    x.x.x           x.x.x         ArkCase Application
```

If the search returns nothing, run `helm repo update` again and retry.

### Optional: Save chart defaults for reference

```bash
# Linux / Mac
helm show values arkcase/app > chart-defaults/app-values.yaml

# Windows PowerShell
helm show values arkcase/app | Out-File -FilePath .\chart-defaults\app-values.yaml -Encoding utf8
```

---

## 11. Deploy ArkCase

### Step 11.1 — Create the Namespace

```bash
kubectl create namespace arkcase --dry-run=client -o yaml | kubectl apply -f -
```

Idempotent — safe to run multiple times.

### Step 11.2 — Install ArkCase

From the root of this repository:

**Linux / Mac:**

```bash
helm upgrade --install arkcase arkcase/app \
  -n arkcase \
  --create-namespace \
  -f values/dev-values.yaml \
  --timeout 20m
```

**Windows PowerShell:**

```powershell
helm upgrade --install arkcase arkcase/app `
  -n arkcase `
  --create-namespace `
  -f values/dev-values.yaml `
  --timeout 20m
```

> **Why `--timeout 20m`?** ArkCase pulls many images on first install. The default 5-minute
> Helm timeout causes a misleading failure even though pods eventually come up.

### Step 11.3 — Watch the Rollout

```bash
kubectl get pods -n arkcase -w
```

`Ctrl+C` when all pods show `Running` and `1/1` in the READY column. First install takes
10–20 minutes.

### Pods stuck in Pending?

```bash
kubectl describe pod <stuck-pod-name> -n arkcase
# Look for "Insufficient memory" or "Insufficient cpu" in Events
```

Ensure `values/dev-values.yaml` has:

```yaml
global:
  dev:
    resources: true   # Reduces resource requests — essential for dev/test environments
```

---

## 12. Read Admin Credentials

ArkCase auto-generates admin credentials and stores them in a Kubernetes Secret.

### Linux / Mac (Bash)

```bash
echo "Username:"
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.username}' | base64 -d; echo

echo "Password:"
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### Windows (PowerShell)

```powershell
$u = kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.username}'
$p = kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($u))
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

### Important: Login username format

The secret stores `arkcase-admin`. The UI requires the domain suffix:

```
arkcase-admin@dev.arkcase.com
```

Using `arkcase-admin` alone returns `?login_error` even with the correct password.

### Optional: Set a Custom Password

**Linux / Mac:**

```bash
kubectl patch secret arkcase-core-main-admin -n arkcase \
  --type merge \
  -p '{"stringData": {"username": "admin", "password": "YourStrongPassword!"}}'

kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

**Windows PowerShell:**

```powershell
$patch = '{"stringData": {"username": "admin", "password": "YourStrongPassword!"}}'
kubectl patch secret arkcase-core-main-admin -n arkcase --type merge -p $patch

kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

---

## 13. Access the App

ArkCase is configured to run at `https://server.dev.arkcase.com/arkcase`.

### Step 13.1 — Find Your Ingress IP / Port

**K3s (NodePort):**

```bash
kubectl get nodes -o wide          # Use INTERNAL-IP column
kubectl get svc -n ingress-nginx   # HTTPS NodePort will be 32443
```

**Minikube:**

```bash
minikube ip
```

**Docker Desktop (Windows):**

The ingress IP is `localhost` / `127.0.0.1` — Docker Desktop routes NodePorts to localhost.

**kubeadm (bare metal — EXTERNAL-IP pending):**

Use port-forward (Step 13.3).

### Step 13.2 — Add Hosts File Entry

**Linux / Mac:**

```bash
# Replace 192.168.x.x with your ingress IP (or 127.0.0.1 for Docker Desktop / port-forward)
echo "192.168.x.x  server.dev.arkcase.com" | sudo tee -a /etc/hosts
```

**Windows** (open Notepad as Administrator, edit `C:\Windows\System32\drivers\etc\hosts`):

```
192.168.x.x  server.dev.arkcase.com
```

Or PowerShell as Administrator:

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "192.168.x.x  server.dev.arkcase.com"
```

### Step 13.3 — Port-Forward (Fallback)

When the ingress IP is not directly routable:

```bash
kubectl port-forward svc/ingress-nginx-controller 8444:443 -n ingress-nginx
```

Update hosts file to use `127.0.0.1`, then access:

```
https://server.dev.arkcase.com:8444/arkcase/
```

### Step 13.4 — Verify the Endpoint

```bash
# Expect 302 (redirect to login)
curl -k -I --resolve server.dev.arkcase.com:443:127.0.0.1 \
  https://server.dev.arkcase.com/arkcase/

# Expect 200
curl -k -I --resolve server.dev.arkcase.com:443:127.0.0.1 \
  https://server.dev.arkcase.com/arkcase/login
```

### Step 13.5 — Log In

Open a browser in Incognito / Private mode (avoids cached session issues):

```
https://server.dev.arkcase.com/arkcase/
```

- **Username:** `arkcase-admin@dev.arkcase.com`
- **Password:** decoded from the secret (Section 12)

---

## 14. Troubleshooting

### `helm: command not found` (Linux)

```bash
ls -la /usr/local/bin/helm
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc && source ~/.bashrc
```

### `helm` not recognized (Windows)

Ensure the directory containing `helm.exe` is in your system `PATH`. Restart PowerShell after changing PATH.

---

### Pods stuck in `Pending`

```bash
kubectl describe pod <pod-name> -n arkcase
# Check the Events section at the bottom
```

| Event message | Cause | Fix |
|---|---|---|
| `Insufficient memory` | Not enough RAM | Increase node memory; ensure `global.dev.resources: true` |
| `Insufficient cpu` | Not enough CPU | Increase CPU allocation |
| `no nodes available that match all predicates` | Control plane taint | `kubectl taint nodes --all node-role.kubernetes.io/control-plane-` |
| `PersistentVolumeClaim is not bound` | No storage provisioner | `kubectl get storageclass` — ensure a default exists |

---

### Pods stuck in `ImagePullBackOff`

```bash
kubectl describe pod <pod-name> -n arkcase
curl -I https://registry-1.docker.io/
```

**K3s proxy config:**

```bash
sudo mkdir -p /etc/systemd/system/k3s.service.d/
cat <<EOF | sudo tee /etc/systemd/system/k3s.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
EOF
sudo systemctl daemon-reload && sudo systemctl restart k3s
```

**containerd proxy config (kubeadm):**

```bash
sudo mkdir -p /etc/systemd/system/containerd.service.d/
cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
EOF
sudo systemctl daemon-reload && sudo systemctl restart containerd
```

---

### Login fails with correct credentials

**Domain suffix missing** — use `arkcase-admin@dev.arkcase.com`, not `arkcase-admin`.

**Pod cached old credentials:**

```bash
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

**Browser cached session** — use a fresh Incognito/Private window or clear cookies for
`server.dev.arkcase.com`.

---

### `kubectl port-forward` fails — address already in use

**Linux:**

```bash
sudo ss -tlnp | grep 8443
kubectl port-forward -n arkcase svc/core 9443:8443   # use alternate port
```

**Windows:**

```powershell
Get-NetTCPConnection -LocalPort 8443 -State Listen
kubectl port-forward -n arkcase svc/core 9443:8443
```

---

### `kubectl get pods -l app.kubernetes.io/name=arkcase` returns nothing

ArkCase uses `instance`, not `name`:

```bash
kubectl get pods -n arkcase -l app.kubernetes.io/instance=arkcase
kubectl get pods -n arkcase   # or just list everything
```

---

### Helm stuck in `pending-install` or `pending-upgrade`

```bash
helm list -A
helm delete arkcase -n arkcase --no-hooks
helm upgrade --install arkcase arkcase/app -n arkcase -f values/dev-values.yaml --timeout 20m
```

---

### Minikube: volume already exists / container not found (RHEL)

```bash
minikube delete --all --purge
podman volume rm minikube 2>/dev/null || true
podman rm -f minikube 2>/dev/null || true
podman network rm minikube 2>/dev/null || true
minikube config unset rootless
unset MINIKUBE_ROOTLESS
minikube start --driver=podman --rootless=false --cpus=6 --memory=12288
```

---

### Minikube: `/run/runc: no such file or directory` (RHEL)

```bash
sudo dnf install -y runc
minikube delete
minikube config unset rootless
unset MINIKUBE_ROOTLESS
minikube start --driver=podman --rootless=false --cpus=6 --memory=12288
```

---

### Docker Desktop: Kubernetes stuck in "Starting" (Windows)

1. Open Docker Desktop → Settings → Kubernetes → **Reset Kubernetes Cluster**
2. Wait 2–3 minutes
3. If still stuck: quit Docker Desktop completely, restart it, wait for the whale icon to
   stabilise, then re-enable Kubernetes

---

## 15. Clean Uninstall & Reinstall

### Uninstall ArkCase

```bash
helm uninstall arkcase -n arkcase
helm list -A
kubectl get pods -n arkcase
```

### Delete Persistent Data (Optional — Destructive)

> ⚠️ This permanently deletes all ArkCase data including the database.

```bash
kubectl get pvc -n arkcase
kubectl delete pvc --all -n arkcase
```

### Reinstall from Scratch

```bash
helm upgrade --install arkcase arkcase/app \
  -n arkcase \
  --create-namespace \
  -f values/dev-values.yaml \
  --timeout 20m

kubectl get pods -n arkcase -w
```

---

## 16. Quick Reference Cheatsheet

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
# Linux/Mac
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.password}' | base64 -d; echo

# ── K3s ───────────────────────────────────────────────────────────
curl -sfL https://get.k3s.io | sh -s - --disable traefik   # install
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER:$USER ~/.kube/config
sudo systemctl status k3s
sudo journalctl -u k3s -n 50

# ── INGRESS ───────────────────────────────────────────────────────
kubectl get svc -n ingress-nginx
kubectl get ingress -n arkcase
kubectl describe ingress -n arkcase
kubectl port-forward svc/ingress-nginx-controller 8444:443 -n ingress-nginx

# ── DEBUGGING ─────────────────────────────────────────────────────
kubectl describe pod <pod-name> -n arkcase
kubectl logs <pod-name> -n arkcase
kubectl logs <pod-name> -n arkcase --previous
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s

# ── CLEANUP ───────────────────────────────────────────────────────
kubectl get pvc -n arkcase
kubectl delete pvc --all -n arkcase   # ⚠️ destroys all data

# ── MINIKUBE RESET (RHEL/Podman) ──────────────────────────────────
minikube delete --all --purge
podman volume rm minikube; podman rm -f minikube; podman network rm minikube
minikube config unset rootless && unset MINIKUBE_ROOTLESS
```

---

## 17. Appendix: values/dev-values.yaml

The `values/dev-values.yaml` in this repo overrides chart defaults for local development.

### Key settings

```yaml
global:
  settings:
    baseUrl: "https://server.dev.arkcase.com/arkcase"
  ingress:
    className: "nginx"   # Use "traefik" only if keeping K3s default ingress
  dev:
    resources: true      # Reduces resource requests — essential for dev/test
```

### Ingress backend annotations

ArkCase's core service speaks HTTPS internally. These annotations tell nginx to proxy correctly:

```yaml
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

### See all chart options

```bash
helm show values arkcase/app | less
```

---

*Guide last updated: March 2026. Covers Ubuntu 20.04–24.04, Debian 11–12, RHEL 8/9,
Rocky Linux 8/9, AlmaLinux 8/9, Fedora 38+, Windows 10/11. K3s v1.30+, Minikube v1.38+,
Kubernetes v1.30, Helm v3.17, Docker Desktop 4.x, ArkCase chart series 25.x.
ArkCase officially tests against vanilla Kubernetes, Rancher Desktop, and EKS.*