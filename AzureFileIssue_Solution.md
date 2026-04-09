# AKS Live PV Mount Options Change — Runbook

> **Objective:** Add missing SMB mount options (`mfsymlinks`, `cache=strict`, `nosharesock`, `nobrl`,
> `actimeo=30`) to existing Azure Files PVs in a production AKS cluster with minimal disruption.
>
> **Context:** The team has already applied some mount options but is missing key performance-related
> ones. Since Kubernetes treats `mountOptions` as **immutable**, PV/PVC must be recreated. This
> runbook provides a safe, step-by-step approach for production.

---

## Table of Contents

1. [Pre-Change Assessment](#1-pre-change-assessment)
2. [Risk & Impact Analysis](#2-risk--impact-analysis)
3. [Pre-Change Checklist](#3-pre-change-checklist)
4. [Change Procedure — Option A: In-Place PV/PVC Recreate (Recommended)](#4-change-procedure--option-a-in-place-pvpvc-recreate)
5. [Change Procedure — Option B: Blue-Green PV Migration](#5-change-procedure--option-b-blue-green-pv-migration)
6. [Post-Change Validation](#6-post-change-validation)
7. [Rollback Plan](#7-rollback-plan)
8. [NFS Migration — Current State & Feasibility](#8-nfs-migration--current-state--feasibility)

---

## 1. Pre-Change Assessment

### 1.1 Identify All Affected PVs and PVCs

```bash
# List all PVCs using Azure Files
kubectl get pvc --all-namespaces -o json | jq -r '
  .items[] |
  select(.spec.storageClassName | test("azurefile|azure-file|file"; "i")) |
  [.metadata.namespace, .metadata.name, .spec.volumeName, .spec.storageClassName] |
  @tsv
' | column -t -s $'\t'
```

### 1.2 Document Current Mount Options

```bash
# For each PV, capture current mount options
PV_NAME=<pv-name>
kubectl get pv $PV_NAME -o jsonpath='{.spec.mountOptions}' | jq .
```

### 1.3 Identify What's Missing

Run this comparison script:

```bash
#!/bin/bash
# file: check-mount-options.sh

REQUIRED_OPTIONS=("dir_mode=0777" "file_mode=0777" "uid=0" "gid=0" "mfsymlinks" "cache=strict" "nosharesock" "nobrl" "actimeo=30")

for PV in $(kubectl get pv -o jsonpath='{.items[*].metadata.name}'); do
  CURRENT=$(kubectl get pv $PV -o jsonpath='{.spec.mountOptions}' 2>/dev/null)

  # Check if it's an Azure Files PV
  DRIVER=$(kubectl get pv $PV -o jsonpath='{.spec.csi.driver}' 2>/dev/null)
  if [[ "$DRIVER" != "file.csi.azure.com" ]]; then
    continue
  fi

  echo "=========================================="
  echo "PV: $PV"
  echo "Current options: $CURRENT"
  echo ""
  echo "Missing options:"
  for OPT in "${REQUIRED_OPTIONS[@]}"; do
    if ! echo "$CURRENT" | grep -q "$OPT"; then
      echo "  ❌ $OPT"
    else
      echo "  ✅ $OPT"
    fi
  done
  echo ""
done
```

### 1.4 Identify Workloads Using Each PVC

```bash
# Find all pods/deployments attached to a specific PVC
PVC_NAME=<pvc-name>
NAMESPACE=<namespace>

echo "=== Pods using PVC: $PVC_NAME ==="
kubectl get pods -n $NAMESPACE -o json | jq -r "
  .items[] |
  select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$PVC_NAME\") |
  \"Pod: \(.metadata.name)  Owner: \(.metadata.ownerReferences[0].kind)/\(.metadata.ownerReferences[0].name)\"
"
```

---

## 2. Risk & Impact Analysis

### Disruption Summary

| Step | Action | Service Impact | Duration |
|------|--------|---------------|----------|
| 1 | Patch PV reclaim policy to Retain | ✅ None | ~5 seconds |
| 2 | Scale down deployment | ⚠️ **Downtime starts** | ~30 seconds |
| 3 | Delete PVC | ✅ None (pods already down) | ~10 seconds |
| 4 | Delete PV | ✅ None (data safe with Retain) | ~10 seconds |
| 5 | Recreate PV with new mount options | ✅ None | ~10 seconds |
| 6 | Recreate PVC | ✅ None | ~10 seconds |
| 7 | Scale up deployment | ⚠️ **Downtime ends** | ~1-2 minutes |
| | **Total downtime** | | **~3-5 minutes per PV** |

### What IS at Risk

| Concern | Risk Level | Mitigation |
|---------|-----------|------------|
| Data loss | 🟢 **Zero** — if reclaim policy set to `Retain` | Step 2 ensures this |
| Azure File Share deletion | 🟢 **Zero** — PV deletion ≠ share deletion with `Retain` | Verified in Step 2 |
| Mount failure after recreate | 🟡 **Low** — if YAML is correct | Pre-validate YAML, have backup ready |
| Application downtime | 🟡 **Expected** — 3-5 min per PV | Schedule during maintenance window |
| Wrong mount options applied | 🟡 **Low** | Validate with node debug pod after |

### What is NOT at Risk

- ✅ Your data on the Azure File Share — it is **completely untouched**
- ✅ Other PVs / PVCs / workloads not being changed
- ✅ The Azure Storage Account and its configuration
- ✅ Snapshots and backup policies on the Azure File Share

---

## 3. Pre-Change Checklist

```
[ ] Identified all PVs that need updating (Section 1.1)
[ ] Documented current mount options for every PV (Section 1.2)
[ ] Identified missing options per PV (Section 1.3)
[ ] Identified all deployments/pods using each PVC (Section 1.4)
[ ] Backed up all PV and PVC YAMLs (Section 4, Step 1)
[ ] Confirmed maintenance window with stakeholders
[ ] Tested the full procedure in non-prod environment
[ ] Prepared updated PV/PVC YAML files (Section 4, Steps 4-5)
[ ] Confirmed rollback plan is ready (Section 7)
[ ] Verified kubectl access and permissions
[ ] Communicated planned downtime to dependent teams
```

---

## 4. Change Procedure — Option A: In-Place PV/PVC Recreate

> **Best for:** Production environments where the same PVC name must be preserved
> (deployment YAMLs don't need to change).
>
> **Downtime:** ~3-5 minutes per PV

### Step 0: Discover Your PV, PVC, and Deployment Details

Before starting, you need to identify the exact PVC name, namespace, PV name, and
deployment name from your AKS cluster. These are **not values you guess** — you
discover them from the running cluster.

```bash
# ─────────────────────────────────────────────────────────────
# STEP 0a: List ALL PVCs across all namespaces
# ─────────────────────────────────────────────────────────────
kubectl get pvc --all-namespaces -o wide

# Example output:
# NAMESPACE     NAME                  STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS          AGE
# production    profiles-pvc          Bound    pv-azurefile01  100Gi      RWX            azurefile-premium     90d
# production    data-pvc              Bound    pv-azurefile02  50Gi       RWX            azurefile-premium     90d
# staging       test-pvc              Bound    pv-azurefile03  20Gi       RWX            azurefile-premium     45d
#
# From this output, note down:
#   NAMESPACE  = "production"         (1st column)
#   PVC_NAME   = "profiles-pvc"       (2nd column)
#   PV_NAME    = "pv-azurefile01"     (4th column — the VOLUME)
```

```bash
# ─────────────────────────────────────────────────────────────
# STEP 0b: Filter ONLY Azure Files PVs and show current mount options
#          This helps you see what's already set and what's missing
# ─────────────────────────────────────────────────────────────
kubectl get pv -o json | jq -r '
  .items[] |
  select(.spec.csi.driver == "file.csi.azure.com") |
  [.metadata.name,
   .spec.claimRef.namespace,
   .spec.claimRef.name,
   (.spec.mountOptions | join(",")),
   .spec.csi.volumeAttributes.shareName] |
  @tsv
' | column -t -s $'\t' -N "PV_NAME,NAMESPACE,PVC_NAME,CURRENT_MOUNT_OPTIONS,AZURE_SHARE"

# Example output:
# PV_NAME          NAMESPACE    PVC_NAME        CURRENT_MOUNT_OPTIONS                  AZURE_SHARE
# pv-azurefile01   production   profiles-pvc    dir_mode=0777,file_mode=0777,uid=0     profilesshare
# pv-azurefile02   production   data-pvc        dir_mode=0777,file_mode=0777           datashare
#
# This tells you:
#   - Which PVs are Azure Files
#   - What mount options they CURRENTLY have (so you can see what's missing)
#   - Which Azure File Share they point to
#   - The namespace and PVC name you need
```

```bash
# ─────────────────────────────────────────────────────────────
# STEP 0c: Find which deployment uses a specific PVC
#          Replace <namespace> and <pvc-name> with values from Step 0a
# ─────────────────────────────────────────────────────────────
kubectl get deployments -n <namespace> -o json | jq -r "
  .items[] |
  select(.spec.template.spec.volumes[]?.persistentVolumeClaim.claimName==\"<pvc-name>\") |
  .metadata.name
"

# Example output:
# profiles-app
#
# This is your DEPLOYMENT_NAME
```

```bash
# ─────────────────────────────────────────────────────────────
# STEP 0d: Set all variables for the rest of this runbook
#          Replace the example values with YOUR actual values
#          discovered from Steps 0a, 0b, and 0c above
# ─────────────────────────────────────────────────────────────

NAMESPACE="production"               # ← From Step 0a (1st column)
PVC_NAME="profiles-pvc"              # ← From Step 0a (2nd column)
PV_NAME="pv-azurefile01"             # ← From Step 0a (4th column / VOLUME)
DEPLOYMENT_NAME="profiles-app"       # ← From Step 0c output

# Verify these are correct before proceeding
echo "=========================================="
echo "Variables set for this change:"
echo "  Namespace:  $NAMESPACE"
echo "  PVC:        $PVC_NAME"
echo "  PV:         $PV_NAME"
echo "  Deployment: $DEPLOYMENT_NAME"
echo "=========================================="
echo ""
echo "PVC Status:"
kubectl get pvc $PVC_NAME -n $NAMESPACE
echo ""
echo "PV Status:"
kubectl get pv $PV_NAME
echo ""
echo "Deployment Status:"
kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE
```

> **⚠️ Important:** All subsequent steps in this runbook use the variables
> `$NAMESPACE`, `$PVC_NAME`, `$PV_NAME`, and `$DEPLOYMENT_NAME`.
> Make sure these are set correctly and verified before proceeding.
>
> If you have **multiple PVs** to update, repeat the entire procedure
> (Steps 0–8) for each PV **one at a time**. Do not change multiple PVs simultaneously.

---

### Step 1: Backup Current PV and PVC Definitions

```bash
# Create backup directory
mkdir -p pv-change-backup/$(date +%Y%m%d)
cd pv-change-backup/$(date +%Y%m%d)

# Backup PV
kubectl get pv $PV_NAME -o yaml > pv-original-${PV_NAME}.yaml

# Backup PVC
kubectl get pvc $PVC_NAME -n $NAMESPACE -o yaml > pvc-original-${PVC_NAME}.yaml

# Backup the deployment
kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE -o yaml > deployment-backup-${DEPLOYMENT_NAME}.yaml

echo "✅ Backups saved:"
ls -la

# Verify backups are not empty
echo ""
echo "PV backup lines: $(wc -l < pv-original-${PV_NAME}.yaml)"
echo "PVC backup lines: $(wc -l < pvc-original-${PVC_NAME}.yaml)"
echo "Deployment backup lines: $(wc -l < deployment-backup-${DEPLOYMENT_NAME}.yaml)"
```

> 🛑 **Do not proceed if any backup file has 0 lines.** Re-run the backup commands.

---

### Step 2: Set Reclaim Policy to Retain

```bash
# ⚠️ THIS IS THE MOST CRITICAL STEP — PROTECTS YOUR DATA
# Without this, deleting the PV could delete the Azure File Share!

# Check current policy
echo "Current reclaim policy:"
kubectl get pv $PV_NAME -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
echo ""

# Patch to Retain
kubectl patch pv $PV_NAME -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# VERIFY — Do not proceed until this shows "Retain"
echo "Updated reclaim policy:"
kubectl get pv $PV_NAME -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
echo ""
# Expected output: Retain
```

> 🛑 **STOP HERE if the output is NOT `Retain`.** Do not proceed until confirmed.

---

### Step 3: Extract PV Details for Recreating

```bash
# Extract all values you'll need for the new PV YAML
echo "==========================================="
echo "PV Details — Copy these for Step 4"
echo "==========================================="
echo "PV Name:          $PV_NAME"
echo "Storage:          $(kubectl get pv $PV_NAME -o jsonpath='{.spec.capacity.storage}')"
echo "Access Modes:     $(kubectl get pv $PV_NAME -o jsonpath='{.spec.accessModes}')"
echo "CSI Driver:       $(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.driver}')"
echo "Volume Handle:    $(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.volumeHandle}')"
echo "Share Name:       $(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.volumeAttributes.shareName}')"
echo "Storage Account:  $(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.volumeAttributes.storageAccount}')"
echo "Secret Name:      $(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.nodeStageSecretRef.name}')"
echo "Secret Namespace: $(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.nodeStageSecretRef.namespace}')"
echo "Current MountOpts:$(kubectl get pv $PV_NAME -o jsonpath='{.spec.mountOptions}')"
echo ""
echo "==========================================="
echo "PVC Details — Copy these for Step 5"
echo "==========================================="
echo "PVC Name:         $PVC_NAME"
echo "Namespace:        $NAMESPACE"
echo "StorageClass:     $(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.storageClassName}')"
```

---

### Step 4: Prepare Updated PV YAML

Create the updated PV YAML — **replace placeholders** with values from Step 3:

```yaml
# file: pv-updated.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <PV_NAME>                                  # ← Same PV name from Step 3
spec:
  capacity:
    storage: 100Gi                                  # ← Same as Step 3 "Storage"
  accessModes:
    - ReadWriteMany                                 # ← Same as Step 3 "Access Modes"
  persistentVolumeReclaimPolicy: Retain
  # ─────────────────────────────────────────────
  #  UPDATED MOUNT OPTIONS (add missing ones)
  # ─────────────────────────────────────────────
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=0
    - gid=0
    - mfsymlinks                                    # ← NEW: Symbolic link support
    - cache=strict                                  # ← NEW: Client-side read caching
    - nosharesock                                   # ← NEW: Dedicated TCP per mount
    - nobrl                                         # ← NEW: Skip byte-range locks
    - actimeo=30                                    # ← NEW: Attribute cache 30 sec
  # ─────────────────────────────────────────────
  csi:
    driver: file.csi.azure.com                      # ← Same as Step 3 "CSI Driver"
    volumeHandle: <VOLUME_HANDLE>                   # ← Same as Step 3 "Volume Handle"
    volumeAttributes:
      shareName: <SHARE_NAME>                       # ← Same as Step 3 "Share Name"
      storageAccount: <STORAGE_ACCOUNT>             # ← Same as Step 3 "Storage Account" (if present)
    nodeStageSecretRef:
      name: <SECRET_NAME>                           # ← Same as Step 3 "Secret Name"
      namespace: <SECRET_NAMESPACE>                 # ← Same as Step 3 "Secret Namespace"
```

---

### Step 5: Prepare Updated PVC YAML

```yaml
# file: pvc-updated.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <PVC_NAME>                                  # ← Same PVC name from Step 3
  namespace: <NAMESPACE>                            # ← Same namespace from Step 3
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: <PV_NAME>                             # ← Bind to specific PV (same name)
  storageClassName: ""                              # ← Empty string for pre-provisioned PV
```

---

### Step 6: Validate YAMLs Before Starting (Dry Run)

```bash
# Dry run — does not create anything, just validates syntax
kubectl apply -f pv-updated.yaml --dry-run=client
kubectl apply -f pvc-updated.yaml --dry-run=client

# Both should show:
# persistentvolume/<name> created (dry run)
# persistentvolumeclaim/<name> created (dry run)

# If either shows errors, fix the YAML before proceeding
```

---

### Step 7: Execute the Change

```bash
# ──────────────────────────────────────────────
# ⏱️ DOWNTIME STARTS HERE
# ──────────────────────────────────────────────

# 7a. Record current replica count
REPLICAS=$(kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE -o jsonpath='{.spec.replicas}')
echo "Current replicas: $REPLICAS — will restore to this after change"

# 7b. Scale down to 0
echo "Scaling down $DEPLOYMENT_NAME..."
kubectl scale deployment $DEPLOYMENT_NAME -n $NAMESPACE --replicas=0

# 7c. Wait for all pods to terminate
echo "Waiting for pods to terminate..."
kubectl wait --for=delete pod -l app=<app-label> -n $NAMESPACE --timeout=120s
echo "✅ All pods terminated"

# 7d. Delete PVC
echo "Deleting PVC..."
kubectl delete pvc $PVC_NAME -n $NAMESPACE
echo "✅ PVC deleted"

# 7e. Delete PV
echo "Deleting PV (Azure File Share is SAFE — Retain policy is set)..."
kubectl delete pv $PV_NAME
echo "✅ PV deleted"

# 7f. Recreate PV with new mount options
echo "Recreating PV with optimized mount options..."
kubectl apply -f pv-updated.yaml
echo "✅ PV recreated"

# 7g. Verify PV is Available
kubectl get pv $PV_NAME
# STATUS must show: Available

# 7h. Recreate PVC
echo "Recreating PVC..."
kubectl apply -f pvc-updated.yaml
echo "✅ PVC recreated"

# 7i. Verify PVC is Bound
kubectl get pvc $PVC_NAME -n $NAMESPACE
# STATUS must show: Bound

# 7j. Scale back up
echo "Scaling up $DEPLOYMENT_NAME to $REPLICAS replicas..."
kubectl scale deployment $DEPLOYMENT_NAME -n $NAMESPACE --replicas=$REPLICAS

# 7k. Wait for pods to be ready
echo "Waiting for pods to be ready..."
kubectl wait --for=condition=ready pod -l app=<app-label> -n $NAMESPACE --timeout=300s
echo "✅ All pods running"

# ──────────────────────────────────────────────
# ⏱️ DOWNTIME ENDS HERE
# ──────────────────────────────────────────────
```

---

### Step 8: Verify New Mount Options Are Active

```bash
# Get the node where the pod is running
POD_NAME=$(kubectl get pods -n $NAMESPACE -l app=<app-label> -o jsonpath='{.items[0].metadata.name}')
NODE_NAME=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')

echo "Pod: $POD_NAME is running on Node: $NODE_NAME"

# Debug into the node
kubectl debug node/$NODE_NAME -it --image=mcr.microsoft.com/cbl-mariner/busybox:2.0 -- chroot /host bash

# Inside the node, check mount options
mount | grep cifs | grep <share-name>

# ✅ Expected output should contain:
# cache=strict,nosharesock,nobrl,actimeo=30,mfsymlinks
```

---

## 5. Change Procedure — Option B: Blue-Green PV Migration

> **Best for:** Environments that need absolute minimum downtime or want to validate
> before cutting over. The old PV stays intact until the new one is verified.
>
> **Downtime:** ~30 seconds (only during deployment update)
>
> **Trade-off:** Requires changing the PVC name in the deployment YAML.

### Step 1: Discover Current Setup

Follow the same **Step 0** from Option A above to identify your `$PV_NAME`,
`$PVC_NAME`, `$NAMESPACE`, and `$DEPLOYMENT_NAME`.

### Step 2: Create a New PV with Updated Options (Pointing to SAME Azure Share)

```yaml
# file: pv-new-optimized.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <PV_NAME>-optimized                         # ← New name (add -optimized suffix)
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
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
  csi:
    driver: file.csi.azure.com
    volumeHandle: <VOLUME_HANDLE>-optimized          # ← Must be unique (add suffix)
    volumeAttributes:
      shareName: <SAME_SHARE_NAME>                   # ← SAME Azure File Share
      storageAccount: <SAME_STORAGE_ACCOUNT>
    nodeStageSecretRef:
      name: <SAME_SECRET_NAME>
      namespace: <SAME_SECRET_NAMESPACE>
```

### Step 3: Create a New PVC Bound to the New PV

```yaml
# file: pvc-new-optimized.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <PVC_NAME>-optimized                         # ← New PVC name
  namespace: <NAMESPACE>
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: <PV_NAME>-optimized                    # ← Bind to new PV
  storageClassName: ""
```

```bash
kubectl apply -f pv-new-optimized.yaml
kubectl apply -f pvc-new-optimized.yaml

# Verify
kubectl get pv <PV_NAME>-optimized                   # Should show: Bound
kubectl get pvc <PVC_NAME>-optimized -n $NAMESPACE   # Should show: Bound
```

### Step 4: Update Deployment to Use New PVC

```bash
# Edit deployment to change PVC name
kubectl edit deployment $DEPLOYMENT_NAME -n $NAMESPACE

# Change:
#   claimName: <PVC_NAME>
# To:
#   claimName: <PVC_NAME>-optimized
```

Kubernetes will perform a **rolling update** — new pods mount the optimized PV,
old pods terminate gracefully. Downtime is near-zero for multi-replica deployments.

### Step 5: Validate and Clean Up

```bash
# After verifying everything works (see Section 6)

# Set reclaim policy to Retain on old PV first
kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Clean up old PV/PVC
kubectl delete pvc <PVC_NAME> -n $NAMESPACE
kubectl delete pv <PV_NAME>
```

---

## 6. Post-Change Validation

### 6.1 Verify Mount Options

```bash
# From node debug pod
mount | grep cifs

# Confirm these options are present:
# ✅ cache=strict
# ✅ nosharesock
# ✅ nobrl
# ✅ actimeo=30
# ✅ mfsymlinks
```

### 6.2 Run IOPS Benchmark

```bash
# From inside the application pod
kubectl exec -it <pod-name> -n <namespace> -- sh

# Quick fio test (install fio if not available)
fio --name=randread --ioengine=libaio --direct=1 --bs=4k \
    --iodepth=64 --rw=randread --numjobs=4 --size=256M \
    --directory=/mnt/<mount-path>/ --group_reporting --runtime=30

fio --name=randwrite --ioengine=libaio --direct=1 --bs=4k \
    --iodepth=64 --rw=randwrite --numjobs=4 --size=256M \
    --directory=/mnt/<mount-path>/ --group_reporting --runtime=30
```

### 6.3 Re-run Prereq Check

Run the same prereq IOPS check that was failing. Expected results after mount option optimization:

| Metric | Before | After (Expected) | Threshold |
|--------|--------|-------------------|-----------|
| Read IOPS | 378 | 800 — 1,500 | 1,800 |
| Write IOPS | 123 | 400 — 800 | 600 |

> **Note:** Mount option optimization alone may get you **close to but not fully past** the read
> threshold. If you still fall short, the next step is the NFS migration (Section 8).

### 6.4 Monitor for 24 Hours

```bash
# Check pod restarts (should be 0)
kubectl get pods -n $NAMESPACE -l app=<app-label> -w

# Check for CIFS errors on the node
kubectl debug node/<node> -it --image=mcr.microsoft.com/cbl-mariner/busybox:2.0 -- \
  chroot /host bash -c "dmesg | grep -i cifs | tail -20"

# Check Azure Storage metrics for throttling
# Azure Portal → Storage Account → Monitoring → Metrics → Transactions → Split by Response Type
```

---

## 7. Rollback Plan

### Which Rollback Applies to You?

| What You Did | Rollback Procedure |
|---|---|
| **Option A** — In-Place PV/PVC Recreate (Section 4) | [7.1 Rollback for Option A](#71-rollback-for-option-a) |
| **Option B** — Blue-Green PV Migration (Section 5) | [7.2 Rollback for Option B](#72-rollback-for-option-b) |
| **NFS Migration** (Section 8) | [7.3 Rollback for NFS Migration](#73-rollback-for-nfs-migration) |

---

### 7.1 Rollback for Option A (In-Place PV/PVC Recreate)

> **When to use:** You completed Section 4 (deleted old PV/PVC, created new ones with
> updated mount options), but something is wrong — pods are crashing, mount is failing,
> or application is not working correctly.
>
> **What this restores:** The original PV/PVC with the **old mount options**
> (before you made any changes).
>
> **Prerequisites:** You must have the backup YAMLs from Step 1 of Section 4.

```bash
# ──────────────────────────────────────────
# ROLLBACK — OPTION A
# ──────────────────────────────────────────

# Navigate to backup directory
cd pv-change-backup/<date-of-change>

# 1. Scale down
kubectl scale deployment $DEPLOYMENT_NAME -n $NAMESPACE --replicas=0
kubectl wait --for=delete pod -l app=<app-label> -n $NAMESPACE --timeout=120s

# 2. Delete the new (broken) PV/PVC
kubectl delete pvc $PVC_NAME -n $NAMESPACE --ignore-not-found
kubectl delete pv $PV_NAME --ignore-not-found

# 3. Clean metadata from backup YAMLs
#    (removes auto-generated fields that Kubernetes won't accept on create)
cat pv-original-${PV_NAME}.yaml | \
  yq 'del(.metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp,
       .metadata.managedFields,
       .metadata.annotations["pv.kubernetes.io/bound-by-controller"],
       .status)' > pv-rollback.yaml

cat pvc-original-${PVC_NAME}.yaml | \
  yq 'del(.metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp,
       .metadata.managedFields,
       .metadata.annotations["pv.kubernetes.io/bind-completed"],
       .metadata.annotations["pv.kubernetes.io/bound-by-controller"],
       .status)' > pvc-rollback.yaml

# 4. Restore original PV
kubectl apply -f pv-rollback.yaml

# 5. Verify PV is Available
kubectl get pv $PV_NAME
# STATUS: Available

# 6. Restore original PVC
kubectl apply -f pvc-rollback.yaml

# 7. Verify PVC is Bound
kubectl get pvc $PVC_NAME -n $NAMESPACE
# STATUS: Bound

# 8. Scale back up
kubectl scale deployment $DEPLOYMENT_NAME -n $NAMESPACE --replicas=$REPLICAS

# 9. Verify pods are running
kubectl wait --for=condition=ready pod -l app=<app-label> -n $NAMESPACE --timeout=300s
kubectl get pods -n $NAMESPACE -l app=<app-label>

echo "✅ Rollback complete — original mount options restored"
```

> **Data impact:** ✅ None — the Azure File Share was never touched.

---

### 7.2 Rollback for Option B (Blue-Green PV Migration)

> **When to use:** You completed Section 5 (created a new `-optimized` PV/PVC and
> updated the deployment to use it), but something is wrong.
>
> **What this restores:** Points the deployment back to the **original PV/PVC**
> (which still exists — that's the advantage of Blue-Green).
>
> **This is the easiest rollback** because the old PV/PVC was never deleted.

```bash
# ──────────────────────────────────────────
# ROLLBACK — OPTION B
# ──────────────────────────────────────────

# 1. Simply revert the deployment to use the original PVC name
kubectl edit deployment $DEPLOYMENT_NAME -n $NAMESPACE

# Change:
#   claimName: <PVC_NAME>-optimized
# Back to:
#   claimName: <PVC_NAME>

# Kubernetes will perform a rolling update back to the original PVC.

# 2. Verify pods are running with original PVC
kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE
kubectl get pods -n $NAMESPACE -l app=<app-label>

# 3. (Optional) Clean up the optimized PV/PVC if you don't need them
kubectl delete pvc <PVC_NAME>-optimized -n $NAMESPACE
kubectl delete pv <PV_NAME>-optimized

echo "✅ Rollback complete — deployment using original PVC again"
```

> **Data impact:** ✅ None — same Azure File Share, just different mount options.
> **Downtime:** Near-zero for multi-replica deployments (rolling update).

---

### 7.3 Rollback for NFS Migration

> **When to use:** You completed Section 8 (switched from SMB to NFS with a new
> file share), but NFS is causing issues — performance, permissions, application
> incompatibility, or you need Azure Backup back.
>
> **What this restores:** Points the deployment back to the **original SMB PV/PVC**.
>
> **Prerequisites:** You kept the old SMB PV/PVC and Azure File Share intact
> (as recommended in Section 8).

```bash
# ──────────────────────────────────────────
# ROLLBACK — NFS MIGRATION
# ──────────────────────────────────────────

# 1. Update deployment to use the original SMB PVC
kubectl edit deployment $DEPLOYMENT_NAME -n $NAMESPACE

# Change:
#   claimName: <PVC_NAME>-nfs        (or whatever the NFS PVC was named)
# Back to:
#   claimName: <PVC_NAME>            (original SMB PVC)

# 2. Wait for rolling update
kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE

# 3. Verify pods are running
kubectl get pods -n $NAMESPACE -l app=<app-label>

# 4. Verify mount is back on SMB
POD_NAME=$(kubectl get pods -n $NAMESPACE -l app=<app-label> -o jsonpath='{.items[0].metadata.name}')
NODE_NAME=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
kubectl debug node/$NODE_NAME -it --image=mcr.microsoft.com/cbl-mariner/busybox:2.0 -- \
  chroot /host bash -c "mount | grep cifs"

# Should show cifs mount (SMB), not nfs4

# 5. ⚠️ IMPORTANT: Sync any data written to NFS share back to SMB share
#    (if the application wrote new data while on NFS)
#    Use a migration pod similar to the one in Section 8

echo "✅ Rollback complete — back on SMB with Azure Backup"
```

> **Data impact:** ⚠️ If data was written while on NFS, you need to sync it back to
> the SMB share. The original SMB data is intact, but any **new** data created
> during the NFS period lives only on the NFS share until synced.

---

### Rollback Decision Quick Reference

```
Something went wrong after the change?
              │
              ▼
  Which change did you make?
      │              │              │
  Option A      Option B       NFS Migration
      │              │              │
      ▼              ▼              ▼
  Restore from   Just revert    Revert deployment
  backup YAMLs   PVC name in    PVC name + sync
  (Section 7.1)  deployment     any new data
                 (Section 7.2)  (Section 7.3)
      │              │              │
      ▼              ▼              ▼
  Downtime:      Downtime:      Downtime:
  ~3-5 min       ~30 sec        ~30 sec + data sync
      │              │              │
      ▼              ▼              ▼
  Data impact:   Data impact:   Data impact:
  ✅ None        ✅ None        ⚠️ Sync needed

```

---

## 8. NFS Migration — Current State & Feasibility

### The Original Question

> "They chose SMB because at that time there was no backup facility for NFS.
> Now they want to know — is backup available for NFS?"

### Answer: Partially

| Feature | SMB (Current) | NFS (Current State — Apr 2026) |
|---------|--------------|-------------------------------|
| **Azure Backup (managed, vaulted)** | ✅ GA | ❌ **Not supported yet** |
| **Manual share snapshots** | ✅ GA | ✅ GA (since Jan 2024) |
| **Snapshot via Portal / CLI / API** | ✅ GA | ✅ GA |
| **Max snapshots** | 200 | 200 |
| **Soft delete** | ✅ GA | ⚠️ Preview |
| **Vaulted backup (offsite, immutable)** | ✅ GA (since 2025) | ❌ Not yet available |
| **Third-party backup (Veeam, Commvault)** | ✅ Supported | ✅ Supported |

### What This Means

- **NFS snapshots work** — you can create / restore / automate them via CLI or API
- **Azure Backup (the managed "Configured" backup in your portal)** still does **NOT support NFS**
- If the team relies on **Azure Backup with Recovery Services vault** (retention policies,
  compliance, geo-redundancy), **they cannot replicate this on NFS today**

### Recommendation

| If the team... | Then... |
|---|---|
| **Requires Azure Backup (vaulted, managed)** | Stay on **SMB**, apply mount option optimizations from this runbook |
| **Can use automated snapshots** (CLI CronJob) as backup | Switch to **NFS** for the IOPS fix |
| **Wants NFS performance + backup** | Use NFS + automate snapshots via Kubernetes CronJob + third-party backup tool |

### NFS Snapshot Automation (If They Choose NFS Later)

```yaml
# file: nfs-snapshot-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nfs-share-snapshot
  namespace: <namespace>
spec:
  schedule: "0 */4 * * *"   # Every 4 hours (adjust as needed)
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot
            image: mcr.microsoft.com/azure-cli:latest
            command:
            - bash
            - -c
            - |
              az login --identity
              SNAPSHOT=$(az storage share snapshot create \
                --account-name <storageaccount> \
                --name <sharename> \
                --query "snapshot" -o tsv)
              echo "✅ Snapshot created: $SNAPSHOT at $(date)"

              # Cleanup snapshots older than 30 days
              CUTOFF=$(date -d "-30 days" +%Y-%m-%dT%H:%M:%S)
              az storage share snapshot list \
                --account-name <storageaccount> \
                --name <sharename> \
                --query "[?snapshot<'$CUTOFF'].snapshot" -o tsv | \
              while read OLD_SNAP; do
                az storage share snapshot delete \
                  --account-name <storageaccount> \
                  --name <sharename> \
                  --snapshot "$OLD_SNAP"
                echo "🗑️ Deleted old snapshot: $OLD_SNAP"
              done
          restartPolicy: Never
```

### Official Documentation References

- NFS Snapshot GA Announcement:
  [https://techcommunity.microsoft.com/blog/azurestorageblog/announcing-the-general-availability-of-nfs-azure-file-share-snapshots/4038596](https://techcommunity.microsoft.com/blog/azurestorageblog/announcing-the-general-availability-of-nfs-azure-file-share-snapshots/4038596)
- Azure Files Snapshot Documentation:
  [https://learn.microsoft.com/en-us/azure/storage/files/storage-snapshots-files](https://learn.microsoft.com/en-us/azure/storage/files/storage-snapshots-files)
- Azure Backup Support Matrix (confirms NFS not supported):
  [https://learn.microsoft.com/en-us/azure/backup/azure-file-share-support-matrix](https://learn.microsoft.com/en-us/azure/backup/azure-file-share-support-matrix)
- NFS Protocol Overview:
  [https://learn.microsoft.com/en-us/azure/storage/files/files-nfs-protocol](https://learn.microsoft.com/en-us/azure/storage/files/files-nfs-protocol)

---

## Quick Decision Matrix

```
                          Do they need Azure Backup (vaulted)?
                                    │
                         ┌──── YES ─┤── NO ────┐
                         │          │           │
                         ▼          │           ▼
                  Stay on SMB       │     Switch to NFS
                  Apply mount       │     (5-8x IOPS boost)
                  option fixes      │     + automate snapshots
                  (Section 4)       │     via CronJob
                         │          │           │
                         ▼          │           ▼
              Does it pass          │     ✅ Problem solved
              IOPS thresholds?      │
                    │               │
             YES ───┤─── NO        │
              │     │     │         │
              ▼     │     ▼         │
           ✅ Done  │  Consider NFS │
                    │  anyway +     │
                    │  third-party  │
                    │  backup       │
                    └───────────────┘
```
