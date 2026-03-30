# ArkCase Kubernetes Repro Guide (Windows + Linux Podman)

This guide gives you a **repeatable, low-stress** way to deploy ArkCase with this repo’s values files.

It includes:
- Windows setup (Docker Desktop + Kubernetes)
- Linux setup (Podman as Minikube driver, or CRI-O + kubeadm for production)
- Debian + RHEL 8 command examples
- Helm + ingress setup
- Admin credential discovery (with optional override)
- Full uninstall + clean reinstall procedure
- Verification checklist

---

## 1) Repository layout

This repo intentionally keeps deployment configuration small:

- `chart-defaults/` (regenerated Helm defaults for ArkCase charts)
- `values/` (runtime overrides used for deploy/redeploy)
- `README-yml-files.md` (full YAML workflow and regeneration instructions)

`values/dev-values.yaml` is the active override file for local development.

`chart-defaults/` is based on chart defaults and can be regenerated at any time from Helm.

For the full defaults/overrides workflow, see: `README-yml-files.md`.

---

## 1.1) How to regenerate chart defaults yourself

After adding/updating the chart repo:

```bash
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update
helm show values arkcase/app > chart-defaults/app-values.yaml
```

Optional (PowerShell, explicit UTF-8 write):

```powershell
helm show values arkcase/app | Out-File -FilePath .\chart-defaults\app-values.yaml -Encoding utf8
```

This gives you raw chart defaults; keep your local runtime overrides in `values/dev-values.yaml`.

---

## 1.2) Golden path (follow in this exact order)

If you want the least-stress, reproducible flow, use this sequence:

1. Install platform prerequisites (Windows or Linux track below).
2. Add Helm repo and (re)generate chart defaults under `chart-defaults/`.
3. Create namespace `arkcase`.
4. Install ingress-nginx (if missing).
5. Install ArkCase into `arkcase` namespace.
6. Read generated admin credentials from `arkcase-core-main-admin`.
7. Configure host mapping + access path (ingress or port-forward).
8. Validate checklist.

---

## 2) Common prerequisites (both OS paths)

Install these CLIs:

- `kubectl`
- `helm` (v3)
- `curl`

Then add and update the ArkCase chart repo:

```bash
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update
```

Verify chart is available:

```bash
helm search repo arkcase/app
```

Create a dedicated namespace for ArkCase workloads:

```bash
kubectl create namespace arkcase --dry-run=client -o yaml | kubectl apply -f -
```

---

## 2.1) Linux package examples (Debian and RHEL 8)

Use these as baseline install commands.

### Debian / Ubuntu

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates gnupg lsb-release

# Podman
sudo apt-get install -y podman

# kubectl (official apt repo recommended by Kubernetes docs)
# Helm (official install script or apt repo)
# Minikube (binary install)
```

> Exact package repo steps for `kubectl`, `helm`, and `minikube` can vary by Debian release. Use official docs for your release if package names differ.

### RHEL 8 / Rocky 8 / Alma 8

> **Important:** Kubernetes does not run directly on Podman. Podman does not expose a CRI socket,
> which kubelet requires. On RHEL, the correct paths are:
> - **Local dev:** use Minikube with the Podman driver (Minikube manages the control plane internally)
> - **Production-like:** use CRI-O + kubeadm (see [Section 4.2](#42-option-b--production-like-kubernetes-with-cri-o))

Install prerequisites:

```bash
sudo dnf -y update
sudo dnf -y install podman curl
```

Install `kubectl`, `helm`, and `minikube` from official binaries:

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# helm (replace 3.17.3 with the current version from https://github.com/helm/helm/releases)
curl -LO https://get.helm.sh/helm-v3.17.3-linux-amd64.tar.gz
tar -zxvf helm-v3.17.3-linux-amd64.tar.gz
mv linux-amd64/helm .
rm -rf linux-amd64 helm-v3.17.3-linux-amd64.tar.gz

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
mv minikube-linux-amd64 minikube

chmod +x kubectl minikube helm
sudo mv kubectl minikube helm /usr/local/bin/
```

Verify tools after install:

```bash
podman --version
kubectl version --client
helm version
minikube version
```

## 2.2) Linux runtime gotchas (Debian + RHEL 8)

### RHEL 8 family

> **Podman is not a Kubernetes container runtime.** Podman does not expose the CRI socket
> that kubelet requires. Use Podman only as the Minikube driver (local dev) or for building
> images. For a real cluster, install CRI-O as the container runtime (see [Section 4.2](#42-option-b--production-like-kubernetes-with-cri-o)).

- If `firewalld` is enabled, ensure local Kubernetes networking is permitted.
- If SELinux policy blocks containers/networking during local testing, check audit logs before changing policy.
- Podman rootless mode is recommended for dev unless your local environment requires rootful networking behavior.

### Debian family

- Ensure cgroup v2/systemd integration is healthy (modern Debian defaults are usually fine).
- If DNS resolution is flaky inside cluster pods, verify host DNS and restart Minikube.

Quick diagnostics:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 3) Path A — Windows (Docker Desktop)

## 3.1 Install Docker Desktop

1. Install Docker Desktop for Windows.
2. Enable:
   - **Use WSL 2 based engine**
   - **Kubernetes** (Settings → Kubernetes → Enable Kubernetes)
3. Wait until Kubernetes status is `Running`.

Check:

```powershell
kubectl cluster-info
kubectl get nodes
```

You should see at least one node (usually `desktop-control-plane`).

## 3.2 Install ingress-nginx (if missing)

```powershell
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace default
kubectl get pods -l app.kubernetes.io/name=ingress-nginx
```

---

## 4) Path B — Linux (Podman + Minikube)

> **How it works:** Kubernetes does not run directly on Podman. Podman does not provide the CRI
> socket that kubelet requires. Minikube solves this by managing the Kubernetes control plane
> inside a Podman-managed container/VM and wiring up a compatible CRI internally.
> **Podman is the driver for Minikube, not the Kubernetes runtime itself.**

There are two sub-paths depending on your use case:

| Goal | Recommended approach |
|------|----------------------|
| Local development | [Option A — Minikube with Podman driver](#41-option-a--local-dev-minikube-with-podman-driver) |
| Production-like cluster | [Option B — CRI-O + kubeadm](#42-option-b--production-like-kubernetes-with-cri-o) |

## 4.1 Option A — Local dev: Minikube with Podman driver

Install Podman, kubectl, helm, and minikube (see [Section 2.1](#21-linux-package-examples-debian-and-rhel-8)) and confirm:

```bash
podman --version
minikube version
kubectl version --client
helm version
```

Start Kubernetes with the Podman driver:

```bash
minikube start --driver=podman --cpus=6 --memory=12288
kubectl get nodes
```

Enable ingress addon:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

## 4.2 Option B — Production-like Kubernetes with CRI-O

If you need a real multi-node Kubernetes cluster on RHEL (not Minikube), use CRI-O as the
container runtime with `kubeadm` or `k3s`. Podman alone is **not sufficient** because it does
not expose the CRI socket that kubelet expects.

High-level steps:

```bash
# 1. Install CRI-O (check https://cri-o.io for current repo setup)
sudo dnf -y install cri-o
sudo systemctl enable --now crio

# 2. Install kubeadm, kubelet, kubectl from the Kubernetes repo
# See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

# 3. Initialize the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# 4. Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

> **Note:** For ingress on a bare-metal/kubeadm cluster, install ingress-nginx via Helm
> instead of using `minikube addons enable ingress`.

---

## 5) ArkCase values used by this repo

`values/dev-values.yaml` currently includes these important overrides:

- Base URL for ingress routing:
  - `global.settings.baseUrl: "https://server.dev.arkcase.com/arkcase"`
- Ingress class + nginx backend protocol annotations:
  - `global.ingress.className: "nginx"`
  - `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`
  - `nginx.ingress.kubernetes.io/ssl-redirect: "true"`

These are required for this chart/cluster pattern because ArkCase backend service is HTTPS.

---

## 6) First install (fresh or repeatable)

From repo root:

```bash
helm upgrade --install arkcase arkcase/app -n arkcase --create-namespace -f values/dev-values.yaml
```

Track rollout:

```bash
kubectl get pods -n arkcase -w
```

When all ArkCase pods are `Running`/`Ready`, continue.

---

## 7) Read generated admin credentials (default behavior)

By default, ArkCase generates admin credentials in secret `arkcase-core-main-admin`.

Important:
- The generated username is often `arkcase-admin` (not `admin`) unless you explicitly override it.
- Copy/paste the password exactly as decoded (it is long and case-sensitive).
- For UI login, this deployment expects a domain-qualified username format:
  - `arkcase-admin@dev.arkcase.com`
  - (plain `arkcase-admin` may return `?login_error` even with the correct password)

Read the active values without changing anything:

### PowerShell

```powershell
$u = kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.username}'
$p = kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($u))
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

### Bash

```bash
kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.password}' | base64 -d; echo
```

> Example from a fresh install in this environment: `arkcase-admin` with a generated random password.

Example login mapping for this environment:
- secret username: `arkcase-admin`
- UI login username: `arkcase-admin@dev.arkcase.com`

If login fails even with decoded values, restart core and retry in a private/incognito browser window:

```bash
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

Optional backend verification (no browser ambiguity):

```bash
# expected success redirect:
# location: /arkcase/home.html#!/welcome
curl -k -i -c arkcase.cookies.txt -b arkcase.cookies.txt \
  --resolve server.dev.arkcase.com:9443:127.0.0.1 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -X POST \
  --data-urlencode "username=arkcase-admin@dev.arkcase.com" \
  --data-urlencode "password=<decoded-secret-password>" \
  https://server.dev.arkcase.com:9443/arkcase/login_post
```

## 7.1) Optional: set a custom admin credential

Only do this if you explicitly want to replace the generated values.

### PowerShell

```powershell
$s = kubectl get secret arkcase-core-main-admin -n arkcase -o json | ConvertFrom-Json
$s.data.username = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('admin'))
$s.data.password = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('StrongPassword123!'))
$s | ConvertTo-Json -Depth 100 | kubectl apply -f -
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

### Bash

```bash
kubectl patch secret arkcase-core-main-admin -n arkcase --type merge -p "$(cat <<'JSON'
{
  \"stringData\": {
    \"username\": \"admin\",
    \"password\": \"StrongPassword123!\"
  }
}
JSON
)"
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

---

## 7.2) Important namespace note

After migration, ArkCase runs in `arkcase` namespace. If a command uses `-n default`, it will fail for ArkCase resources.

Examples:

- ✅ `kubectl port-forward -n arkcase svc/core 8443:8443`
- ❌ `kubectl port-forward -n default svc/core 8443:8443`

---

## 8) Hostname resolution and access

ArkCase ingress host is:

- `server.dev.arkcase.com`

You must map it to your local ingress endpoint.

## 8.1 Windows hosts file

Edit as Administrator:

`C:\Windows\System32\drivers\etc\hosts`

Add one of these:

- If ingress LB IP is reachable:
  - `172.24.0.2 server.dev.arkcase.com`
- If using local tunnel:
  - `127.0.0.1 server.dev.arkcase.com`

## 8.2 Linux hosts file

Edit `/etc/hosts`:

- `127.0.0.1 server.dev.arkcase.com` (with local tunnel)
- or ingress IP if reachable

## 8.3 If direct ingress is not reachable (common on local desktop)

Use local tunnel to ingress controller:

```bash
kubectl port-forward svc/ingress-nginx-controller 8444:443
```

Then browse:

- `https://server.dev.arkcase.com:8444/arkcase/`

Quick check:

```bash
curl -k -I --resolve server.dev.arkcase.com:8444:127.0.0.1 https://server.dev.arkcase.com:8444/arkcase/
curl -k -I --resolve server.dev.arkcase.com:8444:127.0.0.1 https://server.dev.arkcase.com:8444/arkcase/login
```

Expected:
- `/arkcase/` returns `302` redirect to login
- `/arkcase/login` returns `200`

## 8.4 Direct core tunnel option (same URL as before)

If you specifically want to use:

- `https://server.dev.arkcase.com:8443/arkcase/login`

Run:

```bash
kubectl port-forward -n arkcase svc/core 8443:8443
```

If `8443` is already in use on your workstation, use an alternate local port:

```bash
kubectl port-forward -n arkcase svc/core 9443:8443
```

Then browse/test:

```bash
curl -k -I --resolve server.dev.arkcase.com:9443:127.0.0.1 https://server.dev.arkcase.com:9443/arkcase/login
```

Then test:

```bash
curl -k -I --resolve server.dev.arkcase.com:8443:127.0.0.1 https://server.dev.arkcase.com:8443/arkcase/login
```

Expected: `HTTP/1.1 200 OK`

Windows tip (identify who owns local 8443):

```powershell
Get-NetTCPConnection -LocalPort 8443 -State Listen | Select-Object LocalAddress,LocalPort,OwningProcess,State
Get-Process -Id <PID_FROM_ABOVE>
```

---

## 9) Full clean uninstall (current deployment)

This removes ArkCase release resources from the current `arkcase` namespace deployment.

```bash
helm uninstall arkcase -n arkcase
```

Confirm removal:

```bash
helm list -A
kubectl get pods -n arkcase
kubectl get ingress -n arkcase
```

> Note: if you want fully clean storage too, delete leftover PVCs created by the release.

Optional PVC cleanup (destructive):

```bash
kubectl get pvc -n arkcase
kubectl delete pvc <pvc-name> -n arkcase
```

Optional legacy cleanup (only if you previously installed in `default`):

```bash
helm uninstall arkcase -n default
kubectl get pvc -n default
```

---

## 10) Full clean reinstall (certainty test)

1. Reinstall:

```bash
helm upgrade --install arkcase arkcase/app -n arkcase --create-namespace -f values/dev-values.yaml
```

2. Wait for readiness:

```bash
kubectl get pods -n arkcase -w
```

3. Read generated credentials (Section 7), or optionally set custom values (Section 7.1).

  For login, prefer `username@dev.arkcase.com` format (Section 7).

4. Verify ingress object:

```bash
kubectl get ingress arkcase-app -n arkcase -o wide
kubectl describe ingress arkcase-app -n arkcase
```

Expected key points:
- `Ingress Class: nginx`
- host `server.dev.arkcase.com`
- backend service `core:8443`
- nginx annotations include backend protocol HTTPS

5. Validate app endpoint (Section 8).

---

## 11) Verification checklist

- [ ] `helm list -A` shows `arkcase` status `deployed` in namespace `arkcase`
- [ ] All ArkCase pods in namespace `arkcase` are `Running` and `READY 1/1`
- [ ] Pod selector check returns results:
  - `kubectl get pods -n arkcase -l app.kubernetes.io/instance=arkcase`
- [ ] `arkcase-app` ingress in namespace `arkcase` has class `nginx`
- [ ] `curl` check returns `302` and `200` for `/arkcase/` and `/arkcase/login`
- [ ] Login works with:
  - password decoded from `arkcase-core-main-admin`
  - username entered as `<decoded-username>@dev.arkcase.com`

---

## 12) Common failure patterns and fixes

### A) Ingress exists but browser cannot connect

Cause: local host cannot route to ingress controller LB/NodePort.

Fix:
- use `kubectl port-forward svc/ingress-nginx-controller 8444:443`
- set hosts entry to `127.0.0.1 server.dev.arkcase.com`

### B) Login fails with generated or changed credentials

Cause (common):
- `arkcase-core` pod still running old env from secret, or
- username format mismatch (plain username used instead of domain-qualified login).

Fix:

```bash
kubectl rollout restart statefulset/arkcase-core -n arkcase
kubectl rollout status statefulset/arkcase-core -n arkcase --timeout=300s
```

Then retry login with domain-qualified username, e.g.:

- `arkcase-admin@dev.arkcase.com`

### C) Helm upgrade succeeds but pods stuck

Check:

```bash
kubectl get pods -n arkcase
kubectl describe pod <pod-name> -n arkcase
kubectl logs <pod-name> -n arkcase
```

### D) `kubectl get pods -l app.kubernetes.io/name=arkcase` shows no resources

Cause: ArkCase pods are labeled by component name (`core`, `rdbms`, `search`, etc.), not `name=arkcase`.

Fix:

```bash
kubectl get pods -n arkcase -l app.kubernetes.io/instance=arkcase
# or
kubectl get pods -n arkcase -l arkcase.com/task=work
```

### E) `kubectl port-forward -n arkcase svc/core 8443:8443` fails with bind/listen error

Cause: local port `8443` is already occupied (often by an existing `kubectl` tunnel).

Fix:
- stop the existing process using `8443`, or
- use alternate local port `9443:8443` and browse `https://server.dev.arkcase.com:9443/arkcase/login`

---

## 13) Repro command summary (quick run)

```bash
# 1) Tooling + chart defaults
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update
helm show values arkcase/app > chart-defaults/app-values.yaml

# 2) Ingress + namespace
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace default
kubectl create namespace arkcase --dry-run=client -o yaml | kubectl apply -f -

# 3) Deploy ArkCase
helm upgrade --install arkcase arkcase/app -n arkcase --create-namespace -f values/dev-values.yaml

# 4) Read generated admin credentials
kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret arkcase-core-main-admin -n arkcase -o jsonpath='{.data.password}' | base64 -d; echo

# 5) Validate
kubectl get ingress arkcase-app -n arkcase -o wide
```

---

## 14) Clean reinstall script flow (what to run when debugging)

```bash
# Remove current release
helm uninstall arkcase -n arkcase

# Optional legacy cleanup (if you previously used default namespace)
helm uninstall arkcase -n default

# Install in arkcase namespace
helm upgrade --install arkcase arkcase/app -n arkcase --create-namespace -f values/dev-values.yaml

# Wait for pods
kubectl get pods -n arkcase -w

# Read generated credentials (see Section 7)

# Validate ingress + login endpoint
kubectl get ingress arkcase-app -n arkcase -o wide
```

---

If you want, the next step is to run this exact cycle now in your current cluster:
1) uninstall `arkcase`,
2) reinstall with `values/dev-values.yaml`,
3) read generated admin credentials,
4) verify login endpoint.
