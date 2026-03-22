# Phase 1 — Environment Setup

> **Goal:** Install all tools on a fresh Ubuntu 22.04 WSL2 system and create a working 3-node Kubernetes cluster.  
> **Time:** ~2-3 hours  
> **Output:** 3-node kind cluster with nodes in `NotReady` state (intentional — no CNI yet)

---

## Pre-Flight Checks

### Verify WSL2 Version
Run these in **PowerShell** (not WSL):

```powershell
wsl --version
wsl --list --verbose
```

Expected output:
```
NAME            STATE           VERSION
* Ubuntu        Running         2
```

> ❌ If you see `VERSION 1`, run: `wsl --set-version Ubuntu 2`

### Verify Ubuntu & Kernel
Run inside your **WSL Ubuntu terminal**:

```bash
lsb_release -a
```
Expected: `Ubuntu 22.04.x LTS`

```bash
uname -r
```
Expected: `5.15.x-microsoft-standard-WSL2` (minimum kernel version: 5.15)

---

## Step 1: Update System

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget git apt-transport-https ca-certificates gnupg lsb-release
```

---

## Step 2: Install Docker

```bash
# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group (no sudo needed)
sudo usermod -aG docker $USER
newgrp docker
```

### Verify Docker
```bash
docker run hello-world
```
Expected: `Hello from Docker!`

---

## Step 3: Install kubectl

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update && sudo apt install -y kubectl
```

### Verify kubectl
```bash
kubectl version --client
```
Expected: `Client Version: v1.29.x` or newer

---

## Step 4: Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Verify kind
```bash
kind version
```
Expected: `kind v0.22.0 ...` or newer

---

## Step 5: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Verify Helm
```bash
helm version
```

---

## Step 6: Install Networking Tools

```bash
sudo apt install -y tcpdump conntrack iputils-ping iproute2 dnsutils net-tools
```

### Verify
```bash
tcpdump --version
conntrack --version
```

---

## Step 7: Create the Cluster

> ⚠️ **Important:** We pin to `kindest/node:v1.29.2` with a SHA hash.  
> Do NOT use the latest kind node image — it has cgroup incompatibilities with WSL2.

```bash
cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: k8s-net-lab
networking:
  disableDefaultCNI: true
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
nodes:
  - role: control-plane
    image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
  - role: worker
    image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
  - role: worker
    image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
EOF

kind create cluster --config kind-cluster.yaml
```

This takes **3-5 minutes** on first run (downloading the node image).

---

## Step 8: Fix WSL2 kubectl Connectivity

> WSL2 cannot directly reach the Docker bridge network (`172.18.0.0/16`).  
> Direct `kubectl` calls will timeout with TLS errors.  
> **Solution:** Use docker exec as a proxy.

```bash
echo 'alias k="docker exec k8s-net-lab-control-plane kubectl"' >> ~/.bashrc
source ~/.bashrc
```

### Verify the cluster
```bash
k get nodes
```

Expected output:
```
NAME                        STATUS     ROLES           AGE   VERSION
k8s-net-lab-control-plane   NotReady   control-plane   Xs    v1.29.2
k8s-net-lab-worker          NotReady   <none>          Xs    v1.29.2
k8s-net-lab-worker2         NotReady   <none>          Xs    v1.29.2
```

> ✅ `NotReady` is **correct and expected** — there is no CNI installed yet.  
> Nodes cannot communicate until we install Calico in Phase 2.

---

## What You Built

```
WSL2 Ubuntu
└── Docker
    ├── k8s-net-lab-control-plane  (container = k8s node)
    │   └── kube-apiserver, etcd, scheduler, controller-manager
    ├── k8s-net-lab-worker          (container = k8s node)
    └── k8s-net-lab-worker2         (container = k8s node)
```

`disableDefaultCNI: true` means Kubernetes has no networking layer.  
Pods can't be scheduled. Nodes can't communicate. DNS doesn't exist.  
**That's Phase 2's problem.**

---

## Troubleshooting

See [issues.md](issues.md) for:
- `Issue 1` — kind cluster fails with cgroup error
- `Issue 2` — kubectl TLS timeout on WSL2
- `Issue 9` — Stale kubeconfig after cluster recreate

---

➡️ **Next:** [Phase 2 — Calico CNI](phase2-calico-cni.md)
