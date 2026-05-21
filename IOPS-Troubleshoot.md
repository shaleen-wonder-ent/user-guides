# AKS Storage IOPS Troubleshooting Guide

## Table of Contents
1. [Storage Type Overview](#1-storage-type-overview)
2. [Zonal Placement](#2-zonal-placement)
3. [StorageClass Configuration](#3-storageclass-configuration)
4. [VM SKU IOPS Limits](#4-vm-sku-iops-limits)
5. [Disk Size & IOPS Tiers](#5-disk-size--iops-tiers)
6. [Storage Account Throttling](#6-storage-account-throttling)
7. [I/O Queue Depth](#7-io-queue-depth)
8. [CSI Driver Health](#8-csi-driver-health)
9. [Diagnostic Commands](#9-diagnostic-commands)
10. [Decision Tree](#10-decision-tree)

---

## 1. Storage Type Overview

| Storage Type | Protocol | StorageClass | libaio Valid? | Zonal? |
|---|---|---|---|---|
| Azure Managed Disk | Block only | `managed-csi` | ✅ Yes | ✅ Yes (LRS) |
| Azure Managed Disk Premium | Block only | `managed-csi-premium` | ✅ Yes | ✅ Yes (LRS) |
| Azure Files Standard | SMB or NFS | `azurefile-csi` | ❌ No | ✅ Yes (LRS) |
| Azure Files Premium | SMB or NFS | `azurefile-csi-premium` | ❌ No | ✅ Yes (LRS) |
| Azure Files Premium ZRS | SMB or NFS | Custom | ❌ No | ❌ No (ZRS) |

> ⚠️ **Note:** NFS protocol is only supported on **Azure Files Premium (SSD)**, never on Azure Managed Disks.
> ⚠️ **Note:** `libaio` is a block-level I/O engine — only valid for **Azure Managed Disks**, not Azure Files (SMB/NFS).

---

## 2. Zonal Placement

### Why It Matters
Azure Managed Disks and Azure Files (LRS) are **zone-pinned**. If the disk and the AKS node are in **different Availability Zones**, cross-zone I/O occurs — causing latency and IOPS degradation with no error or warning.

```
❌ Cross-Zone (Bad)                ✅ Same Zone (Good)
Zone 1        Zone 2              Zone 1
┌──────────┐  ┌──────────┐        ┌─────────────────────┐
│ AKS Node │  │  Disk    │        │ AKS Node + Disk     │
│  Zone 1  │◄─│  Zone 2  │        │  Zone 1             │
└──────────┘  └──────────┘        └─────────────────────┘
  High latency, IOPS drop ❌         Max IOPS ✅
```

### Zonal Placement Rules

| Scenario | Zonal Placement Applies? |
|---|---|
| Azure Files NFS Premium + LRS | ✅ Yes |
| Azure Files NFS Premium + ZRS | ❌ No (replicated across all zones) |
| Azure Files SMB + LRS | ✅ Yes |
| Azure Files SMB + ZRS | ❌ No |
| Azure Managed Disk (any) | ✅ Yes |

### LRS vs ZRS Trade-off

| Factor | LRS | ZRS |
|---|---|---|
| Zonal placement concern | ✅ Yes | ❌ No |
| `WaitForFirstConsumer` needed | ✅ Yes | Not critical |
| Zone failure resilience | ❌ No | ✅ Yes |
| Cost | Lower | ~25-30% higher |
| Peak IOPS/Latency | ✅ Better | Slightly higher latency |

---

## 3. StorageClass Configuration

### Built-in AKS StorageClasses
AKS built-in StorageClasses (since early 2021) already have `WaitForFirstConsumer` set by default:
```bash
# Verify built-in StorageClass binding mode
kubectl describe storageclass managed-csi | grep VolumeBindingMode
# Expected: VolumeBindingMode: WaitForFirstConsumer
```

### Custom StorageClass — Safe Template

#### For Azure Managed Disk
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-managed-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer   # ← Always set this
allowVolumeExpansion: true
```

#### For Azure Files NFS Premium (LRS)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-azurefile-nfs
provisioner: file.csi.azure.com
parameters:
  protocol: nfs
  skuName: Premium_LRS
mountOptions:
  - nconnect=4                            # ← Multiple TCP connections
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer   # ← Always set this
allowVolumeExpansion: true
```

#### For Azure Files NFS Premium (ZRS) — No Zonal Concern
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-azurefile-nfs-zrs
provisioner: file.csi.azure.com
parameters:
  protocol: nfs
  skuName: Premium_ZRS                    # ← ZRS, no zonal placement issue
mountOptions:
  - nconnect=4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### `volumeBindingMode` Explained

| Mode | Disk Created When | Zone Aware? | PVC Status Before Pod |
|---|---|---|---|
| `Immediate` | PVC is created | ❌ Random zone | `Bound` |
| `WaitForFirstConsumer` | Pod is scheduled | ✅ Same zone as pod | `Pending` |

---

## 4. VM SKU IOPS Limits

The **Node VM SKU** caps IOPS regardless of disk capability.

```
Example:
  Premium SSD Disk = 20,000 IOPS capable
  VM SKU (Standard_D2s_v3) = capped at 3,200 IOPS
  
  Result → Max achievable IOPS = 3,200 ❌
```

### Check Node VM SKU
```bash
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
VM:.metadata.labels."node\.kubernetes\.io/instance-type"
```

### Common VM SKU IOPS Limits

| VM SKU | Max Cached IOPS | Max Uncached IOPS |
|---|---|---|
| Standard_D2s_v3 | 4,000 | 3,200 |
| Standard_D4s_v3 | 8,000 | 6,400 |
| Standard_D8s_v3 | 16,000 | 12,800 |
| Standard_D16s_v3 | 32,000 | 25,600 |
| Standard_D32s_v3 | 64,000 | 51,200 |

> 📘 Full VM IOPS limits: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes

---

## 5. Disk Size & IOPS Tiers

Azure Managed Disk IOPS scale with disk size:

| Disk Size | Max IOPS (Premium SSD) | Max Throughput |
|---|---|---|
| 32 GiB | 120 IOPS | 25 MB/s |
| 128 GiB | 500 IOPS | 100 MB/s |
| 512 GiB | 2,300 IOPS | 150 MB/s |
| 1 TiB | 5,000 IOPS | 200 MB/s |
| 4 TiB | 7,500 IOPS | 250 MB/s |
| 8 TiB | 12,500 IOPS | 480 MB/s |
| 16 TiB | 18,000 IOPS | 750 MB/s |
| 32 TiB | 20,000 IOPS | 900 MB/s |

```bash
# Check PVC size
kubectl get pvc

# Check detailed PVC info
kubectl describe pvc <pvc-name>
```

---

## 6. Storage Account Throttling (Azure Files)

### The Problem — Shared Storage Account
```
❌ Bad — Shared Storage Account:
Azure Storage Account-1 (IOPS shared!)
     ├── File Share 1 (PVC 1) ─── Pod A  ⚠️ IOPS split 3 ways
     ├── File Share 2 (PVC 2) ─── Pod B  ⚠️ IOPS split 3 ways
     └── File Share 3 (PVC 3) ─── Pod C  ⚠️ IOPS split 3 ways

✅ Good — Dedicated Storage Accounts:
Azure Storage Account-1
     └── File Share 1 (PVC 1) ─── Pod A  ✅ Full IOPS
Azure Storage Account-2
     └── File Share 2 (PVC 2) ─── Pod B  ✅ Full IOPS
Azure Storage Account-3
     └── File Share 3 (PVC 3) ─── Pod C  ✅ Full IOPS
```

### Storage Account IOPS Limits

| Tier | Max IOPS |
|---|---|
| Standard | 20,000 IOPS total |
| Premium | 100,000 IOPS total |

### Check if PVCs Share a Storage Account
```bash
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}\
{.spec.csi.volumeAttributes.storageAccount}{"\n"}{end}'
```
> If the **same storage account** appears multiple times → IOPS bottleneck confirmed ❌

---

## 7. I/O Queue Depth

Even with correct zonal placement and large enough disk, if the **application issues single-threaded sequential I/O**, IOPS will be low.

```
App sends 1 I/O at a time          App sends 256 I/Os at a time
         │                                    │
         ▼                                    ▼
  Only 1 IOPS utilized ❌           Disk fully utilized ✅
```

### FIO Benchmark — Correct Way for Azure Files NFS
```bash
fio --name=iops-test \
    --filename=/mnt/data/testfile \
    --rw=randread \
    --bs=4k \
    --iodepth=256 \          # Match to disk's max queue depth
    --runtime=60 \
    --numjobs=4 \
    --ioengine=sync \         # Use sync/psync for NFS, NOT libaio
    --direct=1 \
    --group_reporting
```

### FIO Benchmark — Correct Way for Azure Managed Disk (Block)
```bash
fio --name=iops-test \
    --filename=/mnt/data/testfile \
    --rw=randread \
    --bs=4k \
    --iodepth=256 \
    --runtime=60 \
    --numjobs=4 \
    --ioengine=libaio \       # libaio valid for block storage only
    --direct=1 \
    --group_reporting
```

> ⚠️ **Never use `libaio` for Azure Files (NFS/SMB)** — results will be misleading.

---

## 8. CSI Driver Health

```bash
# Check CSI driver pods are running
kubectl get pods -n kube-system | grep csi

# Check Azure Disk CSI driver logs
kubectl logs -n kube-system -l app=csi-azuredisk-node

# Check Azure File CSI driver logs
kubectl logs -n kube-system -l app=csi-azurefile-node

# Check for volume mount errors on pod
kubectl describe pod <pod-name> | grep -i warning
```

---

## 9. Diagnostic Commands

### Full IOPS Diagnostic Checklist
```bash
# 1. Check node VM SKU and zone
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
VM:.metadata.labels."node\.kubernetes\.io/instance-type",\
ZONE:.metadata.labels."topology\.kubernetes\.io/zone"

# 2. Check all StorageClasses and binding mode
kubectl get storageclass
kubectl describe storageclass <name> | grep VolumeBindingMode

# 3. Check PVC status and size
kubectl get pvc -A

# 4. Check PV and storage account mapping
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}\
{.spec.csi.volumeAttributes.storageAccount}{"\n"}{end}'

# 5. Check pod zone vs disk zone
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'
kubectl get node <node-name> --show-labels | grep topology

# 6. Check for throttling warnings
kubectl describe pod <pod-name> | grep -i "warning\|throttl\|iops"

# 7. Check CSI driver health
kubectl get pods -n kube-system | grep csi
```

### Azure Portal Checks
- Navigate to: **Disk → Metrics → Data Disk IOPS Consumed Percentage**
- Navigate to: **Storage Account → Metrics → Transactions / Throttling**

---

## 10. Decision Tree

```
IOPS Issue Reported
        │
        ▼
Is Storage Type Azure Files or Managed Disk?
        │
        ├── Azure Files (NFS/SMB)
        │         │
        │         ├── LRS or ZRS?
        │         │      ├── LRS → Check Zonal Placement ──► WaitForFirstConsumer set?
        │         │      └── ZRS → Zonal placement not an issue
        │         │
        │         ├── Are multiple PVCs sharing same Storage Account? ──► Split into dedicated accounts
        │         ├── Is libaio being used for benchmark? ──► Switch to fio with ioengine=sync
        │         └── Check nconnect mount option for NFS
        │
        └── Managed Disk (Block)
                  │
                  ├── Check Zonal Placement ──► WaitForFirstConsumer set?
                  ├── Check VM SKU IOPS cap ──► Upgrade VM SKU if needed
                  ├── Check Disk Size ──► Larger disk = higher IOPS tier
                  ├── Check I/O queue depth ──► Increase iodepth in fio
                  └── Check CSI driver health
```

---

## References

- 📘 AKS Storage Concepts: https://learn.microsoft.com/en-us/azure/aks/concepts-storage
- 📘 AKS Storage Best Practices: https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-storage
- 📘 Azure Files Zonal Placement: https://learn.microsoft.com/en-us/azure/storage/files/zonal-placement
- 📘 Azure Managed Disk Performance: https://learn.microsoft.com/en-us/azure/virtual-machines/disks-performance
- 📘 VM SKU IOPS Limits: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes
- 📘 Kubernetes StorageClass Docs: https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode
