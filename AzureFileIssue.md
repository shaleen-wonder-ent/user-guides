# Azure Files SMB — AKS Persistent Volume IOPS Troubleshooting Guide

> **Context:** Azure Premium File Share (100 GiB provisioned, 15.37 GiB used, 179 snapshots) mounted as
> a Persistent Volume in AKS via SMB. IOPS prereq checks are consistently failing:
>
> - Read IOPS: **378** (threshold: 1,800) ❌
> - Write IOPS: **123** (threshold: 600) ❌

---

## 1. What to Check

| # | Check Area | Priority |
|---|------------|----------|
| 1 | Current PV/StorageClass mount options | 🔴 Critical |
| 2 | SMB protocol version on the node | 🔴 Critical |
| 3 | Network path — AKS VNet ↔ Storage Account | 🔴 Critical |
| 4 | Storage account region vs AKS cluster region | 🔴 Critical |
| 5 | AKS node VM size (network bandwidth cap) | 🟡 Medium |
| 6 | Azure Storage metrics — throttling errors | 🟡 Medium |
| 7 | CSI driver version | 🟡 Medium |
| 8 | Snapshot count and overhead | 🟢 Low |
| 9 | Provisioned share size vs required IOPS | 🟢 Low |

---

## 2. How to Check

### 2.1 Current PV / StorageClass Mount Options

```bash
# Get the StorageClass name used by the PVC
kubectl get pvc -n <namespace> -o wide

# Inspect the StorageClass
kubectl get storageclass <storageclass-name> -o yaml

# Inspect the PV directly
kubectl get pv <pv-name> -o yaml | grep -A 15 mountOptions
```

**What to look for:**

```yaml
# ❌ BAD — Missing or minimal mount options
mountOptions: []

# ✅ GOOD — Optimized mount options
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=0
  - gid=0
  - mfsymlinks
  - cache=strict
  - nosharesock
  - nobrl
  - actimeo=30
```

### 2.2 SMB Protocol Version on the Node

SSH into the AKS node (or use a debug pod):

```bash
# Option A: Debug pod on the node
kubectl debug node/<node-name> -it --image=mcr.microsoft.com/cbl-mariner/busybox:2.0
chroot /host

# Check what's actually mounted
mount | grep cifs

# Look for vers= in the output
# ❌ BAD: vers=2.1 or vers=3.0
# ✅ GOOD: vers=3.1.1
```

### 2.3 Network Path — VNet ↔ Storage Account

```bash
# From inside a pod that uses the PVC
kubectl exec -it <pod-name> -n <namespace> -- sh

# Check DNS resolution
nslookup <storageaccount>.file.core.windows.net

# Check latency
ping -c 10 <storageaccount>.file.core.windows.net

# ❌ BAD: Resolves to public IP, latency > 5ms
# ✅ GOOD: Resolves to private IP (10.x.x.x), latency < 2ms
```

```bash
# Check if private endpoint exists (Azure CLI)
az storage account show \
  --name <storageaccount> \
  --resource-group <rg> \
  --query "privateEndpointConnections" -o table

# Check service endpoints on the AKS subnet
az network vnet subnet show \
  --resource-group <rg> \
  --vnet-name <vnet> \
  --name <aks-subnet> \
  --query "serviceEndpoints" -o table
```

### 2.4 Storage Account Region vs AKS Region

```bash
# AKS cluster region
az aks show --name <cluster> --resource-group <rg> --query "location" -o tsv

# Storage account region
az storage account show --name <storageaccount> --resource-group <rg> --query "primaryLocation" -o tsv

# ❌ BAD: Different regions (e.g., eastus vs westus2)
# ✅ GOOD: Same region
```

### 2.5 AKS Node VM Size

```bash
# Check node pool VM size
az aks nodepool list --cluster-name <cluster> --resource-group <rg> \
  --query "[].{Name:name, VMSize:vmSize, Count:count}" -o table

# Check expected network bandwidth for the VM SKU
az vm list-skus --location <region> --size <vm-size> --query "[0].capabilities[?name=='UncachedDiskIOPS' || name=='ExpectedNetworkBandwidth']" -o table
```

> **Note:** Small VM SKUs like `Standard_B2s` or `Standard_D2s_v3` have limited network bandwidth
> (e.g., 1 Gbps) which can bottleneck SMB throughput.

### 2.6 Azure Storage Metrics — Throttling

```bash
# Check for throttling events in the last hour
az monitor metrics list \
  --resource "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>/fileServices/default" \
  --metric "Transactions" \
  --dimension "ResponseType" \
  --interval PT1H \
  --query "value[0].timeseries[?metadatavalues[0].value=='ClientThrottlingError' || metadatavalues[0].value=='ServerBusyError']" \
  -o table
```

Or in the **Azure Portal**:
1. Storage Account → **Monitoring** → **Metrics**
2. Metric: `Transactions`
3. Split by: `Response type`
4. Filter for: `ClientThrottlingError`, `ServerBusyError`

### 2.7 CSI Driver Version

```bash
# Check Azure Files CSI driver version
kubectl get pods -n kube-system -l app=csi-azurefile-node -o jsonpath='{.items[0].spec.containers[0].image}'

# ❌ BAD: v1.28 or older
# ✅ GOOD: v1.30+ (latest stable)
```

### 2.8 Snapshot Count

```bash
# List snapshots and total size
az storage share-rm list \
  --storage-account <storageaccount> \
  --resource-group <rg> \
  --query "[].{Name:name, Quota:shareQuota, UsedBytes:shareUsageBytes}" -o table

# Check snapshot count
az storage share snapshot list \
  --account-name <storageaccount> \
  --name <sharename> \
  --query "length(@)" -o tsv
```

### 2.9 Provisioned Size vs Required IOPS

For **Premium Azure Files**, baseline IOPS is calculated as:

```
Baseline IOPS = MAX(3,000 + (1 × Provisioned GiB), 100)
Burst IOPS    = MAX(10,000, 3 × Baseline IOPS)
```

| Provisioned GiB | Baseline IOPS | Burst IOPS |
|-----------------|---------------|------------|
| 100             | 3,100         | 10,000     |
| 256             | 3,256         | 10,000     |
| 1,024           | 4,024         | 12,072     |

Your **3,100 baseline IOPS** should be more than enough for the 1,800 read + 600 write threshold.
The problem is not provisioned IOPS — it's delivery to the pod.

---

## 3. Reasoning

### Why are you getting 378 Read / 123 Write IOPS instead of 3,100?

```
┌──────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  Pod     │───>│  AKS Node    │───>│  Network Path   │───>│  Azure Storage   │
│          │    │ (SMB Client) │    │  (VNet/PE/SE)   │    │  (Premium Files) │
└──────────┘    └──────────────┘    └─────────────────┘    └──────────────────┘
                 ▲                    ▲                      ▲
                 │                    │                      │
            Bottleneck 1         Bottleneck 2           Bottleneck 3
         Mount options,       Latency, no private     Throttling,
         SMB version,         endpoint, cross-region  snapshot overhead
         VM size limits       routing
```

| Root Cause | Likelihood | Impact on IOPS |
|---|---|---|
| **Suboptimal SMB mount options** (missing `nosharesock`, `nobrl`, `cache=strict`, small buffer sizes) | 🔴 Very High | Can reduce IOPS by 3-5x |
| **No private endpoint** — traffic routing through public internet or NAT gateway | 🔴 High | Adds 5-20ms latency per operation |
| **Old SMB version** (`vers=2.1` or `3.0` instead of `3.1.1`) | 🟡 Medium | 20-40% IOPS reduction |
| **Small AKS node VM** — limited network bandwidth | 🟡 Medium | Caps throughput regardless of storage |
| **SMB protocol overhead on Linux** — encryption, signing, non-native protocol | 🟡 Medium | Inherent ~30-50% overhead vs NFS |
| **High snapshot count (179)** — metadata overhead | 🟢 Low | Minimal direct IOPS impact |

---

## 4. Solutions — Minimal Change to Long-Term

### Phase 1: Quick Wins (Minimal Change) ⏱️ ~30 minutes

#### 1A. Optimize Mount Options on StorageClass

Create a new optimized StorageClass:

```yaml
# file: storageclass-smb-optimized.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-premium-optimized
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
  protocol: smb
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=0
  - gid=0
  - mfsymlinks
  - cache=strict        # Enable client-side caching
  - nosharesock          # Dedicated TCP connection per mount
  - nobrl                # Disable byte range locks (if app doesn't need them)
  - actimeo=30           # Cache file attributes for 30 seconds
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass-smb-optimized.yaml
```

> ⚠️ **Note:** Existing PVCs cannot change StorageClass. You must create a new PVC and migrate data.

**Expected impact:** 🟡 **2-3x IOPS improvement** (378 → 800-1,100 read IOPS)

#### 1B. Ensure Private Endpoint / Service Endpoint Exists

```bash
# Create a private endpoint if one doesn't exist
az network private-endpoint create \
  --name pe-storageaccount \
  --resource-group <rg> \
  --vnet-name <aks-vnet> \
  --subnet <aks-subnet-or-pe-subnet> \
  --private-connection-resource-id "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>" \
  --group-id file \
  --connection-name pe-connection
```

**Expected impact:** 🟡 **1.5-2x IOPS improvement** (reduces latency from ~10-20ms to ~1-2ms)

---

### Phase 2: Medium Effort — Switch to NFS (Recommended Long-Term) ⏱️ ~2-4 hours

This is the **single most impactful change** you can make.

#### 2A. Create NFS StorageClass

```yaml
# file: storageclass-nfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-premium-nfs
provisioner: file.csi.azure.com
parameters:
  protocol: nfs           # Switch from SMB to NFS v4.1
  skuName: Premium_LRS
mountOptions:
  - nconnect=8             # 8 parallel TCP connections (massive perf boost)
  - actimeo=30
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass-nfs.yaml
```

#### 2B. Create New PVC

```yaml
# file: pvc-nfs-profiles.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: profiles-nfs
  namespace: <namespace>
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-premium-nfs
  resources:
    requests:
      storage: 100Gi
```

```bash
kubectl apply -f pvc-nfs-profiles.yaml
```

#### 2C. Migrate Data (Old SMB PV → New NFS PV)

```yaml
# file: migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pv-data-migration
  namespace: <namespace>
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: mcr.microsoft.com/cbl-mariner/busybox:2.0
        command:
          - sh
          - -c
          - |
            echo "Starting migration..."
            cp -av /src/* /dst/
            echo "Migration complete!"
            echo "Source file count: $(find /src -type f | wc -l)"
            echo "Dest file count: $(find /dst -type f | wc -l)"
        volumeMounts:
        - name: smb-source
          mountPath: /src
          readOnly: true
        - name: nfs-dest
          mountPath: /dst
      volumes:
      - name: smb-source
        persistentVolumeClaim:
          claimName: <existing-smb-pvc-name>   # Current SMB PVC
      - name: nfs-dest
        persistentVolumeClaim:
          claimName: profiles-nfs               # New NFS PVC
      restartPolicy: Never
  backoffLimit: 2
```

```bash
kubectl apply -f migration-job.yaml
kubectl logs -f job/pv-data-migration -n <namespace>
```

#### 2D. Update Deployment to Use New NFS PVC

```yaml
# In your deployment spec, change:
volumes:
  - name: profiles
    persistentVolumeClaim:
      claimName: profiles-nfs    # <-- Updated from old SMB PVC name
```

#### Prerequisites for NFS

- ✅ Premium tier (you already have this)
- ✅ Linux nodes (you already have this)
- ⚠️ **Private endpoint or service endpoint required** (NFS does not support public access)
- ⚠️ Storage account must have **Secure transfer disabled** or use a new FileStorage account

```bash
# Verify/update storage account for NFS support
az storage account update \
  --name <storageaccount> \
  --resource-group <rg> \
  --https-only false    # NFS requires this if using existing account

# Or create a new dedicated FileStorage account for NFS
az storage account create \
  --name <newstorageaccount> \
  --resource-group <rg> \
  --location <same-region-as-aks> \
  --sku Premium_LRS \
  --kind FileStorage \
  --https-only false \
  --default-action Deny
```

**Expected impact:** 🟢 **5-8x IOPS improvement** (378 → 2,000-3,100 read IOPS)

---

### Phase 3: Additional Optimizations (If Needed)

| Action | Command | Impact |
|---|---|---|
| Increase provisioned size to 256 GiB | `az storage share-rm update --storage-account <acct> --name <share> --quota 256` | Baseline IOPS: 3,100 → 3,256 |
| Clean up old snapshots | `az storage share snapshot delete --account-name <acct> --name <share> --snapshot <id>` | Reduces metadata overhead, saves cost |
| Upgrade AKS node VM size | `az aks nodepool update --cluster-name <cluster> --name <pool> --resource-group <rg> --node-vm-size Standard_D4s_v3` | More network bandwidth |

---

## Summary — Solution Comparison

| Solution | Effort | Downtime | IOPS Improvement | Passes Threshold? | Long-Term? |
|---|---|---|---|---|---|
| **Optimize SMB mount options** | 🟢 Low | ~5 min (PVC recreate) | 2-3x (~800-1,100) | ⚠️ Maybe | No |
| **Add private endpoint** | 🟢 Low | None | 1.5-2x | ⚠️ Maybe | Partial |
| **SMB optimized + private endpoint** | 🟡 Medium | ~5 min | 3-4x (~1,200-1,500) | ⚠️ Close | Partial |
| **Switch to NFS** ⭐ | 🟡 Medium | ~15 min (migration) | **5-8x (~2,000-3,100)** | ✅ **Yes** | **✅ Yes** |
| **NFS + increased share size** | 🟡 Medium | ~15 min | **8-10x** | ✅ **Yes** | **✅ Yes** |

### ⭐ Recommended Path

```
Immediate  →  Add private endpoint (if missing) + optimize SMB mount options
Short-term →  Switch to NFS (StorageClass change + data migration)
```

The NFS switch is the **definitive fix** — it eliminates the SMB protocol overhead that is the
primary reason your 3,100 baseline IOPS is being delivered as only 378 to the pod.
