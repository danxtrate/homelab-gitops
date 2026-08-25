# Homelab GitOps Services, Power Management & Cloud Backup Runbook

**Date:** August 25, 2026  
**Target Cluster:** Raspberry Pi 5 (`192.168.100.25`) K3s Control Plane  
**GitOps Repository:** [`homelab-gitops`](https://github.com/danxtrate/homelab-gitops)  
**Infrastructure Repository:** [`homelab-infra`](https://github.com/danxtrate/homelab-infra)

---

## 1. Executive Summary & Architecture Overview

Today, we successfully expanded the self-hosted services ecosystem on the **Raspberry Pi 5 central control plane** via declarative **GitOps (ArgoCD)**. We deployed, debugged, and optimized three core workloads:

1. **Homepage**: Central homelab web dashboard with live service tiles, search, and datetime widgets.
2. **UpSnap**: Out-of-band Wake-on-LAN (WoL) and remote shutdown controller for Proxmox VE hypervisors with cryptographic command restrictions.
3. **Vaultwarden**: Zero-knowledge password and secrets manager with native HTTPS WebCrypto support.
4. **Vaultwarden 3-2-1 Cloud Backup Engine**: Automated nightly SQLite snapshot offloaded to Oracle Cloud Infrastructure (OCI) S3.
5. **ArgoCD Control Plane Cloud Backup Engine**: Automated daily export (`argocd-util export`) of all cluster secrets, RBAC, projects, and apps offloaded to OCI S3.

```mermaid
flowchart TD
    subgraph Control_Plane["Raspberry Pi 5 (192.168.100.25)"]
        ArgoCD["ArgoCD Hub Control Plane"]
        Homepage["Homepage Dashboard<br/>(:30030)"]
        UpSnap["UpSnap WoL & Power<br/>(:30090 / :8090)"]
        Vaultwarden["Vaultwarden<br/>(HTTPS / :443)"]
        CronBackup["Vaultwarden Backup CronJob<br/>(Daily @ 03:00 AM)"]
    end

    subgraph Proxmox_Nodes["Proxmox VE Hypervisors"]
        PVE1["Node 1 (192.168.100.150)"]
        PVE2["Node 2 (192.168.100.180)"]
        PVE3["Node 3 (192.168.100.190)"]
    end

    subgraph Cloud_Offsite["Oracle Cloud Infrastructure (OCI)"]
        S3["OCI S3 Bucket: homelab<br/>(eu-frankfurt-1)"]
    end

    ArgoCD -->|Syncs Manifests| Homepage
    ArgoCD -->|Syncs Manifests| UpSnap
    ArgoCD -->|Syncs Manifests| Vaultwarden
    ArgoCD -->|Syncs Manifests| CronBackup

    UpSnap -->|UDP 9 Magic Packet (Wake)| Proxmox_Nodes
    UpSnap -->|SSH command=/sbin/poweroff (Shutdown)| Proxmox_Nodes

    CronBackup -->|Non-destructive SQLite Snapshot| Vaultwarden
    CronBackup -->|SigV4 Encrypted S3 Upload| S3
```

---

## 2. Workload Breakdown & Technical Implementations

### A. Central Dashboard: Homepage
* **Repository Path:** [`workloads/homepage/`](../workloads/homepage/)
* **Access URL:** `http://192.168.100.25:30030`
* **ArgoCD Application:** `apps/homepage.yaml`

#### Key Engineering Solutions:
1. **Next.js Host Validation**: Resolved `Host validation failed` by explicitly defining `HOMEPAGE_ALLOWED_HOSTS: "192.168.100.25:30030,192.168.100.25,localhost:3000,localhost"`.
2. **Kubernetes Read-Only Volume Pattern**: Solved `EROFS: read-only file system` on `/app/config` and `/app/config/logs` by implementing an **`initContainer` (busybox)** that copies GitOps ConfigMap files into an ephemeral, fully writable **`emptyDir: {}`** volume.
3. **UI Optimization**: Set `hideErrors: true` in `settings.yaml` to suppress transient container metric popups while preserving categorized service tiles (Proxmox, OPNsense, QNAP, ArgoCD, UpSnap, Vaultwarden, Jellyfin, Immich, Home Assistant).

---

### B. Wake-on-LAN & Power Manager: UpSnap
* **Repository Path:** [`workloads/upsnap/`](../workloads/upsnap/)
* **Access URL:** `http://192.168.100.25:30090` (and host port `8090`)
* **ArgoCD Application:** `apps/upsnap.yaml`

#### Key Engineering Solutions:
1. **Host Network Integration**: Configured `hostNetwork: true` and `dnsPolicy: ClusterFirstWithHostNet` allowing the container to broadcast raw UDP port 9 magic packets across `192.168.100.0/24`.
2. **Deployment Strategy**: Enforced `strategy.type: Recreate` to eliminate host port binding conflicts (`Address already in use`) and `local-path` RWO volume locks during rolling updates.
3. **Security-Hardened Remote Shutdown**:
   * Stored a dedicated ED25519 private key in `/app/pb_data/id_upsnap` (mode `0600`).
   * Created and committed the automation script [`scripts/setup_upsnap_shutdown_keys.sh`](../../homelab-infra/scripts/setup_upsnap_shutdown_keys.sh) in `homelab-infra`.
   * Enforced OpenSSH command locking on all Proxmox hosts (`192.168.100.150`, `.180`, `.190`):
     ```text
     command="/sbin/poweroff",no-port-forwarding,no-agent-forwarding,no-X11-forwarding,no-pty ssh-ed25519 ...
     ```
   * Configured UpSnap shutdown command template using dynamic substitution:
     ```bash
     ssh -i /app/pb_data/id_upsnap -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@{{ DEVICE_IP }} poweroff
     ```

---

### C. Password & Secrets Manager: Vaultwarden
* **Repository Path:** [`workloads/vaultwarden/`](../workloads/vaultwarden/)
* **Access URL:** `https://192.168.100.25` *(Traefik HTTPS)*
* **ArgoCD Application:** `apps/vaultwarden.yaml`

#### Key Engineering Solutions:
1. **HTTPS WebCrypto Ingress**: Deployed a Traefik Ingress resource on entrypoint `websecure` (port 443) with `DOMAIN: "https://192.168.100.25"` to satisfy Bitwarden’s zero-knowledge client-side encryption (`window.crypto.subtle`) requirements.
2. **Storage Durability**: Bound to a 1Gi `local-path` PersistentVolumeClaim (`vaultwarden-data`) for SQLite database (`/data/db.sqlite3`) and RSA cryptographic keys.

---

### D. Automated 3-2-1 OCI S3 Backup Engine
* **Manifest Path:** [`workloads/vaultwarden/backup-cronjob.yaml`](../workloads/vaultwarden/backup-cronjob.yaml)
* **Schedule:** Daily at 03:00 AM UTC (`0 3 * * *`)
* **Destination:** `s3://homelab/backups/vaultwarden/` (OCI Object Storage in `eu-frankfurt-1`)

#### Workflow:
1. Mounts `vaultwarden-data` as **`readOnly: true`** (ensuring zero data modification risk).
2. Executes a transactionally safe live SQLite snapshot:
   ```bash
   sqlite3 /data/db.sqlite3 ".backup /tmp/b/db.sqlite3"
   ```
3. Bundles database, RSA keys, and attachments into a compressed archive `vaultwarden_YYYYMMDD_HHMMSS.tar.gz`.
4. Transmits the payload to OCI Object Storage using an inline Python AWS Signature Version 4 (SigV4) script using credentials from Kubernetes Secret `oci-s3-backup-secret`.

---

### E. Automated ArgoCD Control Plane Backup Engine
* **Manifest Path:** [`workloads/argocd/backup-cronjob.yaml`](../workloads/argocd/backup-cronjob.yaml)
* **Schedule:** Daily at 04:00 AM UTC (`0 4 * * *`)
* **Destination:** `s3://homelab/backups/argocd/` (OCI Object Storage in `eu-frankfurt-1`)

#### Workflow:
1. Runs with `serviceAccountName: argocd-server` to export all ArgoCD custom resource definitions, secrets, repositories, clusters, and configmaps using `argocd-util export -n argocd`.
2. Packages the stream into a compressed archive `argocd_backup_YYYYMMDD_HHMMSS.yaml.gz`.
3. Transmits the payload to Oracle Cloud Infrastructure S3 using AWS SigV4 Python script.

#### Disaster Recovery / Restore Process:
If the Raspberry Pi 5 hardware or storage ever fails:
```bash
# 1. Fresh K3s + ArgoCD install on new drive
curl -sfL https://get.k3s.io | sh - && sudo k3s kubectl create namespace argocd && sudo k3s kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Restore latest backup from OCI S3
gunzip -c argocd_backup_latest.yaml.gz | sudo k3s kubectl exec -i -n argocd deploy/argocd-server -- argocd-util import -n argocd -
```

---

## 3. Quick Reference One-Liners

### Check Pod & Service Status
```bash
ssh pi@192.168.100.25 "sudo k3s kubectl get pods,svc,ingress,cronjob -A | grep -E 'homepage|upsnap|vaultwarden|argocd'"
```

### Trigger On-Demand Vaultwarden S3 Backup
```bash
ssh pi@192.168.100.25 "sudo k3s kubectl create job --from=cronjob/vaultwarden-s3-backup manual-vault-backup-\$(date +%s) -n vaultwarden && sudo k3s kubectl logs -n vaultwarden -l job-name -f"
```

### Trigger On-Demand ArgoCD S3 Backup
```bash
ssh pi@192.168.100.25 "sudo k3s kubectl create job --from=cronjob/argocd-s3-backup manual-argo-backup-\$(date +%s) -n argocd && sudo k3s kubectl logs -n argocd -l job-name -f"
```

### Copy OCI S3 Secret from Vaultwarden to ArgoCD Namespace
```bash
ssh pi@192.168.100.25 "sudo k3s kubectl create secret generic oci-s3-backup-secret -n argocd --from-literal=AWS_ACCESS_KEY_ID=\"\$(sudo k3s kubectl get secret oci-s3-backup-secret -n vaultwarden -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)\" --from-literal=AWS_SECRET_ACCESS_KEY=\"\$(sudo k3s kubectl get secret oci-s3-backup-secret -n vaultwarden -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)\" --dry-run=client -o yaml | sudo k3s kubectl apply -f -"
```

### Run UpSnap Proxmox Key Setup on a Specific Host
```bash
~/Documents/GitOps/homelab-infra/scripts/setup_upsnap_shutdown_keys.sh 192.168.100.150
```

---

## 4. Repository Manifest Tree

```text
homelab-gitops/
├── apps/
│   ├── argocd-backup.yaml
│   ├── homepage.yaml
│   ├── upsnap.yaml
│   └── vaultwarden.yaml
├── docs/
│   └── gitops-services-and-backup-runbook.md
└── workloads/
    ├── argocd/
    │   ├── backup-cronjob.yaml
    │   └── kustomization.yaml
    ├── homepage/
    │   ├── configmap.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    ├── upsnap/
    │   ├── pvc.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── vaultwarden/
        ├── pvc.yaml
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── backup-cronjob.yaml
        └── kustomization.yaml
```
