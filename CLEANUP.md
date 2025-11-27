# Cleanup & Reset Guide 🧹

This guide will help you completely remove the K3s cluster, Argo CD, and all associated tools from your system, returning it to a clean state.

## ⚠️ Warning
**This will delete all data in your Kubernetes cluster.** Make sure you have backed up anything important.

---

## 1. Uninstall K3s
K3s comes with a built-in uninstall script that removes the cluster and system services.

```bash
# Run the uninstall script
/usr/local/bin/k3s-uninstall.sh
```

## 2. Remove Leftover Data & Configs
The uninstall script might leave some configuration files and data directories behind.

```bash
# Remove K3s configuration and data
sudo rm -rf /etc/rancher
sudo rm -rf /var/lib/rancher
sudo rm -rf /var/lib/kubelet
sudo rm -rf /etc/cni
sudo rm -rf /opt/cni

# Remove kubeconfig
rm -rf $HOME/.kube
```

## 3. Uninstall Tools (Optional)
If you want to remove the CLI tools we installed:

### Remove Cilium CLI
```bash
sudo rm /usr/local/bin/cilium
```

### Remove Helm
```bash
sudo rm /usr/local/bin/helm
rm -rf $HOME/.helm
rm -rf $HOME/.cache/helm
rm -rf $HOME/.config/helm
```

### Remove Cloudflared
```bash
sudo apt remove cloudflared
sudo rm -rf /etc/cloudflared
rm -rf $HOME/.cloudflared
```

## 4. Cloudflare Cleanup (Manual)
You should manually clean up resources on Cloudflare to avoid stale entries.

1.  **Delete the Tunnel**:
    -   Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/).
    -   Navigate to **Networks** > **Tunnels**.
    -   Find your tunnel (e.g., `k3s-cluster`) and delete it.
2.  **Delete DNS Records**:
    -   Go to your Domain DNS settings.
    -   Remove any CNAME records pointing to the tunnel (e.g., `argocd.basilaslam.com`).

## 5. Remove System Packages (Optional)
If you want to remove the system packages we installed (only do this if you are sure you don't need them for other things):

```bash
sudo apt remove zfsutils-linux nfs-kernel-server cifs-utils open-iscsi
sudo apt autoremove
```

---

## Verification
To verify everything is gone:

```bash
# Should return "command not found"
k3s --version
kubectl get nodes
cilium version
```

Your system is now clean! ✨
