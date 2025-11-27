# K3s Argo CD GitOps Setup 🚀

This repository contains a **production-ready GitOps configuration** for a K3s cluster on AWS EC2, managed by Argo CD.

It is designed to be **simple to set up** but **powerful enough for production**, featuring:
- **Argo CD** for GitOps deployment.
- **Cilium** for advanced networking and security.
- **Cloudflare Tunnels** for secure external access (no open ports required!).
- **Monitoring Stack** (Prometheus, Grafana) included.

---

## 📚 Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: System Preparation](#step-1-system-preparation)
4. [Step 2: Install K3s](#step-2-install-k3s)
5. [Step 3: Networking (Cilium)](#step-3-networking-cilium)
6. [Step 4: Security & Secrets](#step-4-security--secrets)
7. [Step 5: Deploy Everything (The Magic Step)](#step-5-deploy-everything-the-magic-step)
8. [How to Extend](#how-to-extend)

---

## Architecture Overview

The setup is organized into three main "ApplicationSets". Think of these as groups of applications that Argo CD manages for you.

1.  **Infrastructure** (Sync Wave -5): Core components like Networking, Cert Manager, and Argo CD itself. Deploys first.
2.  **Monitoring** (Sync Wave 0): Tools to watch your cluster (Prometheus, Grafana).
3.  **Apps** (Sync Wave 5): Your actual business applications. Deploys last.

### Directory Structure
```text
.
├── sets/                   # 🚀 ENTRY POINT: Apply these files to start!
│   ├── infrastructure.yaml
│   ├── monitoring.yaml
│   └── apps.yaml
├── infrastructure/         # System components (Cilium, Cloudflared, etc.)
├── monitoring/             # Observability tools
└── apps/                   # Your applications go here
```

---

## Prerequisites

- An **AWS EC2 Instance** (Ubuntu 22.04 or 24.04 recommended).
    - Instance Type: `t3.medium` or larger recommended (2 vCPU, 4GB RAM).
- A **Cloudflare Account** (Free tier is fine) for DNS and Tunnels.
- Basic familiarity with the command line.

---

## Step 1: System Preparation

SSH into your EC2 instance. We need to install some basic tools and prepare the kernel for our networking plugin (Cilium).

```bash
# 1. Update your system and install essential packages
sudo apt update && sudo apt install -y \
  zfsutils-linux \
  nfs-kernel-server \
  cifs-utils \
  open-iscsi \
  curl \
  unzip

# 2. Load critical kernel modules for Cilium Networking
sudo modprobe iptable_raw xt_socket
echo -e "xt_socket\niptable_raw" | sudo tee /etc/modules-load.d/cilium.conf

# 3. Install AWS CLI (Optional, but helpful for finding IPs)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm awscliv2.zip
```

---

## Step 2: Install K3s

K3s is a lightweight Kubernetes distribution. We need to install it with specific flags to disable things we will replace with better alternatives (like replacing Flannel with Cilium).

### 🔍 Find your IP Addresses
You need your **Private IP** for the K3s node setup.

**Option A: From AWS Console**
1. Go to EC2 Dashboard > Instances.
2. Select your instance.
3. Look for **Private IPv4 address** (e.g., `172.31.x.x`).

**Option B: From Terminal**
```bash
# Get Private IP (Use this for SETUP_NODEIP)
hostname -I | awk '{print $1}'
```

### 2. K3s Installation
```bash
# Customize these values!
export SETUP_NODEIP=172.31.7.100  # Your AWS Private IP
export SETUP_PUBLICIP=65.1.210.12 # Your AWS Public IP
export SETUP_CLUSTERTOKEN=randomtokensecret12343  # Strong token
# =====================

curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.33.3+k3s1" \
  INSTALL_K3S_EXEC="--node-ip $SETUP_NODEIP \
  --tls-san $SETUP_PUBLICIP \
  --disable=flannel,local-storage,metrics-server,servicelb,traefik \
  --flannel-backend='none' \
  --disable-network-policy \
  --disable-cloud-controller \
  --disable-kube-proxy" \
  K3S_TOKEN=$SETUP_CLUSTERTOKEN \
  K3S_KUBECONFIG_MODE=644 sh -s -

# Configure kubectl so you can run commands
mkdir -p $HOME/.kube && sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config && chmod 600 $HOME/.kube/config

# Verify it works (Status should be 'NotReady' because we haven't installed networking yet)
kubectl get nodes
```

### 🔌 Optional: Access from Local Machine
Want to control the cluster from your laptop?

1.  **Copy the Kubeconfig**:
    ```bash
    # Run this on your LOCAL machine
    scp ubuntu@65.1.210.12:~/.kube/config ~/.kube/config-k3s
    ```
2.  **Update the Server Address**:
    -   Open `~/.kube/config-k3s` in a text editor.
    -   Replace `https://127.0.0.1:6443` with `https://65.1.210.12:6443`.
3.  **Use it**:
    ```bash
    export KUBECONFIG=~/.kube/config-k3s
    kubectl get nodes
    ```
> **⚠️ Important**: You must ensure **Port 6443** (TCP) is open in your AWS Security Group (Inbound Rules) for your IP address.

---

## Step 3: Networking (Cilium)

Now we install Cilium, which handles all the networking magic.

```bash
# 1. Install Cilium CLI tool
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64 && [ "$(uname -m)" = "aarch64" ] && CLI_ARCH=arm64
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz*

# 2. Install Helm (Package manager for Kubernetes)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 3. Install Cilium into the cluster
helm repo add cilium https://helm.cilium.io && helm repo update
helm install cilium cilium/cilium -n kube-system \
  -f infrastructure/networking/cilium/values.yaml \
  --version 1.18.0 \
  --set operator.replicas=1

# 4. Wait for it to be ready
echo "Waiting for Cilium..."
cilium status --wait
```

---

## Step 4: Security & Secrets

We use **Cloudflare** to securely expose your apps to the internet without opening ports on your firewall.

### 1. Get Cloudflare Credentials
1.  Go to your [Cloudflare Dashboard](https://dash.cloudflare.com/).
2.  **API Token**:
    -   Go to **My Profile** > **API Tokens**.
    -   Create Token > Use template **Edit zone DNS**.
    -   Select your domain.
    -   Copy the token.

### 2. Create the Secrets in Kubernetes
Run these commands in your terminal, replacing the placeholders.

```bash
# === SET YOUR CREDENTIALS ===
export CLOUDFLARE_API_TOKEN="your-api-token-here"
export CLOUDFLARE_EMAIL="your-email@example.com"
# ============================

# Create namespace and secret for Cert Manager
kubectl create namespace cert-manager
kubectl create secret generic cloudflare-api-token -n cert-manager \
  --from-literal=api-token=$CLOUDFLARE_API_TOKEN \
  --from-literal=email=$CLOUDFLARE_EMAIL
```

### 3. Setup Cloudflare Tunnel
This connects your cluster to Cloudflare.

```bash
# 1. Install cloudflared tool
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# 2. Login (This will give you a URL to visit in your browser)
cloudflared tunnel login

# 3. Create the tunnel
export TUNNEL_NAME="k3s-cluster"
cloudflared tunnel create $TUNNEL_NAME

# 4. Create the secret in Kubernetes
kubectl create namespace cloudflared
# We assume the creds file was created in ~/.cloudflared/
kubectl create secret generic tunnel-credentials \
  --namespace=cloudflared \
  --from-file=credentials.json=$HOME/.cloudflared/*.json

# 5. Clean up local credentials for security
rm $HOME/.cloudflared/*.json
```

---

## Step 5: Deploy Everything (The Magic Step)

Now that the foundation is laid, we need to **bootstrap** Argo CD so it can take over.

### 1. Bootstrap Argo CD
Since Argo CD manages itself, we need to install it manually first (the "Chicken and Egg" problem).

```bash
# 1. Create the namespace
kubectl create namespace argocd

# 2. Add the Argo Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 3. Install Argo CD (Bootstrap)
helm install argocd argo/argo-cd -n argocd \
  -f infrastructure/controllers/argocd/values.yaml \
  --version 7.3.6

# 4. Wait for it to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

### 2. Apply the GitOps Config
Now that Argo CD is running, we can tell it to manage the whole cluster.

```bash
# Apply the Projects (Security Boundaries)
kubectl apply -f infrastructure/controllers/argocd/projects/

# Apply the ApplicationSets
kubectl apply -f sets/infrastructure.yaml
kubectl apply -f sets/monitoring.yaml
kubectl apply -f sets/apps.yaml
```

**What happens next?**
1.  **Infrastructure** apps (Argo CD, Cert Manager, etc.) will start syncing.
2.  Once those are healthy, **Monitoring** apps will start.
3.  Finally, your **Apps** (Backend, Frontend) will deploy.

You can watch the progress:
```bash
kubectl get applications -n argocd -w
```

---

## Step 6: Configure DNS 🌐

Since you have a **Wildcard CNAME** record (`*.basilaslam.com`) pointing to your tunnel, **you do not need to add individual records!** 🎉

The `cloudflared` tunnel is configured to send all traffic for `*.basilaslam.com` to the internal Cilium Gateway, which then routes it to the correct app based on the hostname.

**Verify your Wildcard Record:**
1.  Go to **Cloudflare Dashboard** > **DNS**.
2.  Ensure you have:

| Type | Name | Target | Proxy Status |
| :--- | :--- | :--- | :--- |
| CNAME | `*` | `<your-tunnel-id>.cfargotunnel.com` | Proxied (Orange Cloud) |

That's it! You can immediately access:
-   **Argo CD**: `https://argocd.basilaslam.com`
-   **Backend**: `https://backend.basilaslam.com`
-   **Frontend**: `https://frontend.basilaslam.com`

---

## How to Extend

### Adding a New App
1.  Create a folder: `apps/my-new-app`.
2.  Put your Kubernetes YAML files (Deployment, Service, Ingress) in there.
3.  Git Commit and Push.
4.  Argo CD sees the new folder and deploys it automatically!

### Adding a New Infrastructure Tool
1.  Create a folder: `infrastructure/controllers/my-tool`.
2.  Add your Helm values or YAML manifests.
3.  Git Commit and Push.

---

## Troubleshooting

-   **Node Not Ready?** Check if Cilium is running: `kubectl get pods -n kube-system`.
-   **Argo CD not syncing?** Check the logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server`.
-   **Cloudflare Tunnel issues?** Check the tunnel logs: `kubectl logs -n cloudflared -l app=cloudflared`.
