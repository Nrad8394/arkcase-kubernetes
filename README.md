# ArkCase on Kubernetes — Complete RHEL Guide

> **Who this is for:** Anyone deploying ArkCase via Helm on RHEL 8/9, Rocky Linux, or AlmaLinux.
> **What's fixed vs the original README:** The original had incorrect Podman/Kubernetes assumptions,
> no RHEL system prerequisites, a broken Helm install command, and no guidance on what ArkCase
> actually tests against. All of that is corrected here.

---

## Table of Contents

1. [Before You Start — Read This](#1-before-you-start--read-this)
2. [System Requirements](#2-system-requirements)
3. [Choose Your Path](#3-choose-your-path)
4. [Path A — K3s ⭐ Recommended](#4-path-a--k3s--recommended)
5. [Path B — Minikube (Single Machine, More Complex)](#5-path-b--minikube-single-machine-more-complex)
6. [Path C — kubeadm (Production Multi-Node)](#6-path-c--kubeadm-production-multi-node)
7. [Install Helm](#7-install-helm)
8. [Add the ArkCase Helm Repo](#8-add-the-arkcase-helm-repo)
9. [Deploy ArkCase](#9-deploy-arkcase)
10. [Read Admin Credentials](#10-read-admin-credentials)
11. [Access the App](#11-access-the-app)
12. [Troubleshooting](#12-troubleshooting)
13. [Clean Uninstall & Reinstall](#13-clean-uninstall--reinstall)
14. [Quick Reference Cheatsheet](#14-quick-reference-cheatsheet)
15. [Appendix: values/dev-values.yaml](#15-appendix-valuesdev-valuesyaml)

---

## 1. Before You Start — Read This

### What ArkCase Actually Tests Against

ArkCase's official Helm charts are tested against **vanilla Kubernetes**, **Rancher Desktop**,
and **EKS**. Minikube, K3s, and OpenShift are listed as "should work" — but are not officially
validated. This matters because every problem you hit with an untested stack is yours to debug,
not a documented issue with a known fix.

**K3s is the closest you can get to "vanilla Kubernetes" on a single RHEL machine**, which is
why it is the recommended path in this guide.

### The Podman Misconception (Original README Was Wrong)

The original README implied you could use Podman as a Kubernetes runtime. **This does not work.**

- **Podman** is a container tool. It has no CRI (Container Runtime Interface) socket, which
  is what Kubernetes (`kubelet`) requires to manage containers.
- **Minikube with the Podman driver** is different — Minikube runs Kubernetes *inside* a
  Podman-managed container. Podman is the *VM driver*, not the runtime. The actual runtime
  inside Minikube is containerd or CRI-O.
- On RHEL 9 with rootless Podman, Minikube hits additional bugs: broken DNS, `/run/runc`
  access errors, and stale volume state after crashes. These are real, documented issues — not
  user error.

### Quick Decision Tree

```
What is your goal?
  ├─ Test/develop ArkCase on one machine (RHEL 8/9)
  │    └─ Use Path A: K3s ← Start here. Two commands to a working cluster.
  │
  ├─ Already have Minikube set up and want to continue
  │    └─ Use Path B: Minikube ← Read the caveats carefully.
  │
  └─ Production or multi-node cluster
       └─ Use Path C: kubeadm + CRI-O
```

---

## 2. System Requirements

### Minimum Hardware

| Component    | K3s / Minikube (single node)       | kubeadm (per node)      |
|--------------|------------------------------------|-------------------------|
| CPU cores    | 6                                  | 4 (8+ recommended)      |
| RAM          | 12 GB                              | 16 GB                   |
| Disk         | 50 GB free                         | 100 GB                  |
| OS           | RHEL 8/9, Rocky 8/9, AlmaLinux 8/9 | Same                    |

> **Why so much RAM?** ArkCase is a full enterprise stack. Every component runs as its own pod:
> the core app, PostgreSQL/MariaDB, Solr search, Alfresco content, ActiveMQ messaging, Samba LDAP,
> and Pentaho reporting. Under-resourcing is the single most common cause of pods stuck in `Pending`.

### Software Prerequisites

- RHEL 8, RHEL 9, Rocky Linux 8/9, or AlmaLinux 8/9
- A user with `sudo` privileges
- Internet access (or a private registry with ArkCase images pre-seeded)

---

## 3. Choose Your Path

| Path | Best for | Runtime | Complexity |
|------|----------|---------|------------|
| **A — K3s** ⭐ | Dev/test on one RHEL machine | containerd (built in) | Low |
| **B — Minikube** | If you specifically need Minikube | containerd / CRI-O inside VM | Medium–High on RHEL 9 |
| **C — kubeadm** | Real multi-node cluster | CRI-O | High |

All paths converge at [Section 7 — Install Helm](#7-install-helm).

---

## 4. Path A — K3s ⭐ Recommended

K3s is a lightweight, fully conformant Kubernetes distribution packaged as a single binary.
It bundles containerd as its runtime, requires zero driver configuration, and works cleanly
on RHEL 8/9 out of the box. It is the lowest-friction path to a working ArkCase cluster on
a single machine.

### Step 4.1 — System Prep

#### Disable swap

Kubernetes requires swap to be disabled:

```bash
sudo swapoff -a
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab

# Verify (should return nothing)
swapon --show
```

#### Set SELinux to Permissive

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Verify
getenforce
# Expected: Permissive
```

> SELinux permissive mode still logs violations — it just does not block them. This is the
> same approach used by OpenShift and is required for Kubernetes networking to function on RHEL.

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

#### Open firewall ports

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp    # K3s API server
sudo firewall-cmd --permanent --add-port=10250/tcp   # kubelet
sudo firewall-cmd --permanent --add-port=443/tcp     # HTTPS ingress
sudo firewall-cmd --permanent --add-port=80/tcp      # HTTP ingress
sudo firewall-cmd --reload
```

### Step 4.2 — Install K3s

```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

That is the entire install. K3s will download its binary, set up containerd, start the
Kubernetes control plane, and create a systemd service that starts on boot. The
`--disable traefik` flag skips K3s's default ingress so we can install nginx ingress instead
(which matches ArkCase's chart defaults).

Wait a few seconds, then verify:

```bash
sudo k3s kubectl get nodes
# Expected:
# NAME        STATUS   ROLES                  AGE   VERSION
# localhost   Ready    control-plane,master   30s   v1.x.x
```

### Step 4.3 — Configure kubectl Access

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

# Verify
kubectl get nodes
```

### Step 4.4 — Install kubectl (Standalone Binary)

K3s includes `kubectl` internally, but installing the standalone binary lets you use it
without `sudo`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

kubectl version --client
```

### Step 4.5 — Install ingress-nginx

First install Helm (jump to [Section 7](#7-install-helm) and come back), then:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.https=32443

# Wait for the ingress controller pod to become ready
kubectl get pods -n ingress-nginx -w
# Ctrl+C when STATUS shows Running
```

Now continue to [Section 8 — Add the ArkCase Helm Repo](#8-add-the-arkcase-helm-repo).

---

## 5. Path B — Minikube (Single Machine, More Complex)

> ⚠️ **Read this before choosing Minikube on RHEL 9:**
> Minikube on RHEL 9 with Podman has several documented issues — rootless DNS failures,
> `/run/runc` permission errors, stale volume state after crashes. If you hit these,
> the fix is usually to delete everything and restart. K3s (Path A) avoids all of these.
> Choose Minikube only if you have a specific reason to.

### Step 5.1 — System Prep

Same as K3s — disable swap, set SELinux permissive, load kernel modules, set sysctl params.
Run the commands in [Step 4.1](#step-41--system-prep) first.

### Step 5.2 — Install Podman and runc

```bash
sudo dnf -y update
sudo dnf -y install podman runc

podman --version
runc --version
```

### Step 5.3 — Force Rootful Mode

Rootless Podman breaks Minikube networking on RHEL 9. Unset it and force rootful:

```bash
# Remove any MINIKUBE_ROOTLESS setting from shell profiles
grep -r MINIKUBE_ROOTLESS ~/.bashrc ~/.bash_profile ~/.profile 2>/dev/null
# If found, remove those lines

unset MINIKUBE_ROOTLESS

# Set up passwordless sudo for Podman
sudo visudo
# Add this line at the bottom (replace yourusername with your actual username):
# yourusername ALL=(ALL) NOPASSWD: /usr/bin/podman
```

### Step 5.4 — Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install -o root -g root -m 0755 minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

minikube version
```

### Step 5.5 — Clean Any Broken State

If you have a failed previous install:

```bash
minikube delete --all --purge
podman volume rm minikube 2>/dev/null || true
podman rm -f minikube 2>/dev/null || true
podman network rm minikube 2>/dev/null || true
minikube config unset rootless
```

### Step 5.6 — Start Minikube

```bash
minikube start \
  --driver=podman \
  --rootless=false \
  --cpus=6 \
  --memory=12288

kubectl get nodes
```

### Step 5.7 — Enable Ingress

```bash
minikube addons enable ingress

kubectl get pods -n ingress-nginx -w
# Ctrl+C when all pods show Running
```

> **Do not** also install ingress-nginx via Helm alongside the Minikube addon. The original
> README mixes both and causes conflicts. Use the addon only.

Now continue to [Section 7 — Install Helm](#7-install-helm).

---

## 6. Path C — kubeadm (Production Multi-Node)

Run all steps on **every node** unless marked "control plane only".

### Step 6.1 — System Prep (All Nodes)

```bash
sudo dnf -y update
sudo dnf -y install curl wget tar

# Disable swap
sudo swapoff -a
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab

# SELinux permissive
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

#### Firewall — Control Plane

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250-10252/tcp
sudo firewall-cmd --reload
```

#### Firewall — Worker Nodes

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

### Step 6.2 — Install CRI-O (All Nodes)

```bash
KUBERNETES_VERSION=v1.30

cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

sudo dnf -y install cri-o
sudo systemctl enable --now crio
sudo systemctl status crio
```

### Step 6.3 — Install Kubernetes Components (All Nodes)

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

### Step 6.4 — Initialize the Cluster (Control Plane Only)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> ⚠️ The output prints a `kubeadm join` command. **Copy it now.** You need it for Step 6.6.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
# Shows NotReady — normal until CNI is installed next
```

### Step 6.5 — Install CNI (Control Plane Only)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

kubectl get nodes -w
# Ctrl+C when all nodes show Ready
```

### Step 6.6 — Join Worker Nodes (Workers Only)

Run the `kubeadm join` command printed in Step 6.4:

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Verify from control plane:

```bash
kubectl get nodes
```

### Step 6.7 — Install Ingress Controller (Control Plane Only)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

kubectl get pods -n ingress-nginx -w
```

---

## 7. Install Helm

> Run on the machine where you execute `helm` and `kubectl` (same machine for K3s/Minikube,
> control plane for kubeadm).

```bash
HELM_VERSION=v3.17.3
curl -LO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"
tar -zxvf "helm-${HELM_VERSION}-linux-amd64.tar.gz"

# NOTE: install sets permissions on the binary itself, not the directory
sudo install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm

rm -rf linux-amd64 "helm-${HELM_VERSION}-linux-amd64.tar.gz"
```

Verify:

```bash
helm version
```

> **`helm: command not found` after install?** Fix PATH:
> ```bash
> echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
> source ~/.bashrc
> ```

---

## 8. Add the ArkCase Helm Repo

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
mkdir -p chart-defaults
helm show values arkcase/app > chart-defaults/app-values.yaml
```

---

## 9. Deploy ArkCase

### Step 9.1 — Create the Namespace

```bash
kubectl create namespace arkcase --dry-run=client -o yaml | kubectl apply -f -
```

This is idempotent — safe to run multiple times, will not error if the namespace already exists.

### Step 9.2 — Install ArkCase

From the root of this repository:

```bash
helm upgrade --install arkcase arkcase/app \
  -n arkcase \
  --create-namespace \
  -f values/dev-values.yaml \
  --timeout 20m
```

> **Why `--timeout 20m`?** The original README omits a timeout. ArkCase pulls many images on
> first install, and the default 5-minute Helm timeout causes a misleading failure even though
> pods eventually come up. 20 minutes is a safe buffer.

### Step 9.3 — Watch the Rollout

```bash
kubectl get pods -n arkcase -w
```

Press `Ctrl+C` when all pods show `Running` and `1/1` in the READY column. First install
typically takes 10–20 minutes.

### Pods stuck in Pending?

Almost always a resource issue:

```bash
kubectl describe pod <stuck-pod-name> -n arkcase
# Look for "Insufficient memory" or "Insufficient cpu" in Events
```

Make sure `values/dev-values.yaml` has:

```yaml
global:
  dev:
    resources: true   # Reduces resource requests for dev/test environments
```

---

## 10. Read Admin Credentials

ArkCase auto-generates admin credentials and stores them in a Kubernetes Secret.

### Bash

```bash
echo "Username:"
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.username}' | base64 -d; echo

echo "Password:"
kubectl get secret arkcase-core-main-admin -n arkcase \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### PowerShell (Windows)

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

Using `arkcase-admin` without the domain returns `?login_error` even with the correct password.

### Optional: Set a custom password

```bash
kubectl patch secret arkcase-core-main-admin -n arkcase \
  --type merge \
  -p '{
    "stringData": {
      "username": "admin",
      "password": "YourStrongPasswordHere!"
    }
  }'

kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

---

## 11. Access the App

ArkCase is configured to run at `https://server.dev.arkcase.com/arkcase`.
Your machine needs to resolve that hostname to your ingress controller.

### Step 11.1 — Find Your Ingress IP / Port

**K3s with NodePort ingress:**

```bash
# Node IP
kubectl get nodes -o wide
# Use the INTERNAL-IP column

# NodePort for HTTPS (will be 32443 if you followed Step 4.5)
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

**Minikube:**

```bash
minikube ip
```

**kubeadm (if EXTERNAL-IP shows `<pending>`):**

Use port-forward — see Step 11.3.

### Step 11.2 — Add a Hosts File Entry

**Linux:**

```bash
# Replace 192.168.x.x with your actual ingress IP
echo "192.168.x.x  server.dev.arkcase.com" | sudo tee -a /etc/hosts
```

**Windows** (`C:\Windows\System32\drivers\etc\hosts`, open as Administrator):

```
192.168.x.x  server.dev.arkcase.com
```

### Step 11.3 — Port-Forward (Fallback)

When the ingress IP is not routable from your machine:

```bash
kubectl port-forward svc/ingress-nginx-controller 8444:443 -n ingress-nginx
```

Update hosts file to:

```
127.0.0.1  server.dev.arkcase.com
```

Then access at `https://server.dev.arkcase.com:8444/arkcase/`

### Step 11.4 — Verify the Endpoint

```bash
# Expect HTTP 302 (redirect to login)
curl -k -I --resolve server.dev.arkcase.com:443:127.0.0.1 \
  https://server.dev.arkcase.com/arkcase/

# Expect HTTP 200
curl -k -I --resolve server.dev.arkcase.com:443:127.0.0.1 \
  https://server.dev.arkcase.com/arkcase/login
```

### Step 11.5 — Log In

Open a browser in Incognito/Private mode (avoids cached session issues):

```
https://server.dev.arkcase.com/arkcase/
```

- **Username:** `arkcase-admin@dev.arkcase.com`
- **Password:** decoded from the secret (Section 10)

---

## 12. Troubleshooting

### `helm: command not found`

```bash
ls -la /usr/local/bin/helm

echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

---

### Pods stuck in `Pending`

```bash
kubectl describe pod <pod-name> -n arkcase
```

| Event message | Cause | Fix |
|---|---|---|
| `Insufficient memory` | Not enough RAM | Increase node memory or set `global.dev.resources: true` |
| `Insufficient cpu` | Not enough CPU | Increase node CPU |
| `no nodes available that match all predicates` | Control plane taint | `kubectl taint nodes --all node-role.kubernetes.io/control-plane-` |
| `PersistentVolumeClaim is not bound` | No storage provisioner | `kubectl get storageclass` — ensure a default exists |

---

### Pods stuck in `ImagePullBackOff`

```bash
kubectl describe pod <pod-name> -n arkcase
curl -I https://registry-1.docker.io/
```

If behind a corporate proxy, configure the K3s service:

```bash
sudo mkdir -p /etc/systemd/system/k3s.service.d/
cat <<EOF | sudo tee /etc/systemd/system/k3s.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
EOF
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

---

### Login fails with correct credentials

**Domain suffix missing** — use `arkcase-admin@dev.arkcase.com`, not `arkcase-admin`.

**Pod cached old credentials** — restart core:

```bash
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

**Browser cached session** — use a fresh Incognito window or clear cookies for
`server.dev.arkcase.com`.

---

### `kubectl port-forward` fails — address already in use

```bash
sudo ss -tlnp | grep 8443

# Use an alternate local port instead
kubectl port-forward -n arkcase svc/core 9443:8443
```

---

### `kubectl get pods -l app.kubernetes.io/name=arkcase` returns nothing

ArkCase pods use `instance`, not `name`:

```bash
kubectl get pods -n arkcase -l app.kubernetes.io/instance=arkcase
# or simply
kubectl get pods -n arkcase
```

---

### Helm stuck in `pending-install` or `pending-upgrade`

```bash
helm list -A
helm delete arkcase -n arkcase --no-hooks
helm upgrade --install arkcase arkcase/app -n arkcase -f values/dev-values.yaml --timeout 20m
```

---

### Minikube: volume already exists / container not found

Stale Minikube state. Clean everything:

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

### Minikube: `/run/runc: no such file or directory`

Rootless Podman cannot access the runc socket:

```bash
# Ensure runc is installed
sudo dnf install -y runc

# Force rootful mode (more reliable)
minikube delete
minikube config unset rootless
unset MINIKUBE_ROOTLESS
minikube start --driver=podman --rootless=false --cpus=6 --memory=12288
```

---

## 13. Clean Uninstall & Reinstall

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

## 14. Quick Reference Cheatsheet

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

# ── K3s ───────────────────────────────────────────────────────────
# Install (no Traefik)
curl -sfL https://get.k3s.io | sh -s - --disable traefik
# Kubeconfig
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER:$USER ~/.kube/config
# Status / logs
sudo systemctl status k3s
sudo journalctl -u k3s -n 50

# ── INGRESS ───────────────────────────────────────────────────────
kubectl get svc -n ingress-nginx
kubectl get ingress -n arkcase
kubectl describe ingress -n arkcase
# Port-forward when IP not directly routable
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

# ── MINIKUBE RESET (when broken) ──────────────────────────────────
minikube delete --all --purge
podman volume rm minikube
podman rm -f minikube
podman network rm minikube
minikube config unset rootless
```

---

## 15. Appendix: values/dev-values.yaml

The `values/dev-values.yaml` in this repo overrides chart defaults for local development.

### Key settings

```yaml
global:
  settings:
    baseUrl: "https://server.dev.arkcase.com/arkcase"
  ingress:
    className: "nginx"      # Change to "traefik" only if keeping K3s default ingress
  dev:
    resources: true         # Reduces resource requests — essential for dev machines
```

### Ingress backend annotations

ArkCase's core service speaks HTTPS. These annotations tell nginx to proxy to it correctly:

```yaml
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

### See all chart options

```bash
helm show values arkcase/app | less
```

---

*Guide last updated: March 2026. Covers RHEL 8/9, Rocky Linux 8/9, K3s v1.30+, Minikube v1.38+,
Kubernetes v1.30, Helm v3.17, ArkCase chart series 25.x. ArkCase officially tests against
vanilla Kubernetes, Rancher Desktop, and EKS. K3s is the recommended single-node path for RHEL.*
