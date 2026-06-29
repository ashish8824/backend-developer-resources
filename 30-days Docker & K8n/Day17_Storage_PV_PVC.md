# Day 17: Kubernetes Storage — PersistentVolumes, PersistentVolumeClaims & StorageClasses

**Goal for today:** Understand how Kubernetes abstracts storage away from Pods, what PersistentVolumes (PV), PersistentVolumeClaims (PVC), and StorageClasses are, how the request-provision-bind-mount cycle works, and how to run stateful workloads like PostgreSQL in Kubernetes with real persistent storage. By the end, you should be able to provision storage for any workload and explain the full storage abstraction layer clearly.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 15: Services — stable networking, load balancing, DNS
- Day 16: ConfigMaps & Secrets — externalising config and sensitive data

**Today's connecting thought:**
> In Docker (Day 5), you used named volumes for persistence. Kubernetes has the same concept but with a critical addition — an abstraction layer that separates **what storage is needed** (the claim) from **what storage exists** (the volume). This lets developers ask for storage without knowing whether it's backed by an AWS EBS volume, a GCP Persistent Disk, NFS, or local disk. The storage admin (or StorageClass) handles that. Today we learn exactly how this works.

---

## 1. The Problem — Why Pods Can't Own Storage Directly

### Pod storage is ephemeral by default

```
Pod starts → container writes /data/myfile.txt
Pod crashes → Kubernetes reschedules Pod on a DIFFERENT node
New Pod starts → /data/myfile.txt is GONE
(the file was on the old node's disk — new node has nothing)
```

This is fine for stateless apps. But for a PostgreSQL database, your entire data is in `/var/lib/postgresql/data` — losing it on every restart is catastrophic.

### The Docker volumes approach doesn't scale

Docker named volumes work on a single machine — the volume lives on the host. In Kubernetes, Pods can be rescheduled to **any node** in the cluster. A volume tied to one node's disk is useless if the Pod moves to another node.

**What you need:**
- Storage that is independent of any specific node
- Storage that persists across Pod restarts and rescheduling
- Storage that Pods can request without knowing the underlying infrastructure

**This is exactly what PersistentVolumes solve.**

---

## 2. The Three Core Storage Objects

Kubernetes storage has three distinct objects — understanding all three and how they relate is the key to understanding Kubernetes storage.

```
┌─────────────────────────────────────────────────────────────────────┐
│  StorageClass                                                        │
│  "Recipe for provisioning storage"                                   │
│  Defined by cluster admin                                           │
│  Example: aws-ebs-gp3, azure-disk, nfs-storage                     │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ used by
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PersistentVolume (PV)                                               │
│  "Actual storage resource in the cluster"                            │
│  Represents a real piece of storage (EBS vol, NFS share, etc.)      │
│  Cluster-scoped (not namespace-scoped)                              │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ bound to
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PersistentVolumeClaim (PVC)                                         │
│  "Request for storage by a Pod"                                      │
│  Created by developer/namespace                                      │
│  Namespace-scoped                                                   │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ mounted by
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Pod                                                                 │
│  Mounts the PVC like a directory                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Teaching analogy — the hotel room analogy:**
> "A StorageClass is the hotel's room type catalogue (Standard, Deluxe, Suite). A PersistentVolume is an actual room in the hotel. A PersistentVolumeClaim is a guest's room request ('I need a Deluxe room for 2 nights'). The hotel (Kubernetes) matches the request to a room and checks the guest in. The Pod is the guest using the room."

---

## 3. PersistentVolume (PV) — The Actual Storage

### WHAT
A **PersistentVolume** represents a piece of storage in the cluster — an AWS EBS volume, a GCP Persistent Disk, an NFS share, a local disk, etc. It is a cluster-level resource (not namespace-scoped) provisioned either manually by a cluster admin or automatically by a StorageClass.

### WHY
PVs abstract the underlying storage technology. A Pod doesn't need to know it's using an EBS volume — it just mounts a PV. If the cluster migrates from EBS to Ceph, only the PV definition changes — Pod specs stay the same.

### HOW — PV YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  # ── Capacity ──────────────────────────────────────────────────────────────
  capacity:
    storage: 20Gi

  # ── Access Modes ──────────────────────────────────────────────────────────
  accessModes:
    - ReadWriteOnce       # one node can mount read-write at a time

  # ── Reclaim Policy ────────────────────────────────────────────────────────
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle

  # ── Storage Class ─────────────────────────────────────────────────────────
  storageClassName: standard

  # ── Volume Source (what backs this PV) ────────────────────────────────────
  # Option A: AWS EBS volume
  awsElasticBlockStore:
    volumeID: vol-0a1b2c3d4e5f6789
    fsType: ext4

  # Option B: Local disk (node-specific — use only with StatefulSets)
  # local:
  #   path: /mnt/data
  # nodeAffinity:
  #   required:
  #     nodeSelectorTerms:
  #       - matchExpressions:
  #           - key: kubernetes.io/hostname
  #             operator: In
  #             values: [worker-node-1]

  # Option C: NFS
  # nfs:
  #   server: 192.168.1.100
  #   path: /exports/postgres

  # Option D: HostPath (for local dev / Minikube only)
  # hostPath:
  #   path: /mnt/data
  #   type: DirectoryOrCreate
```

### Access Modes — Critical to Understand

| Access Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted by ONE node for read-write. Most common. Used by databases. |
| `ReadOnlyMany` | ROX | Mounted by MANY nodes for read-only. |
| `ReadWriteMany` | RWX | Mounted by MANY nodes for read-write. Requires NFS or cloud file storage. |
| `ReadWriteOncePod` | RWOP | Mounted by ONE pod only (strictest). K8s 1.22+. |

**Important nuance:** Access modes control node-level access, not Pod-level. RWO means one **node** — multiple Pods on the same node can all mount it.

```
RWO (most storage: EBS, local disk)
  ✅ One node mounts read-write
  ❌ Cannot be used by Pods on different nodes simultaneously

RWX (NFS, Azure Files, EFS)
  ✅ Multiple nodes mount simultaneously
  ✅ Works for shared file storage across Pods on different nodes
```

### Reclaim Policies

| Policy | What happens when PVC is deleted |
|---|---|
| `Retain` | PV keeps the data — must be manually reclaimed by admin |
| `Delete` | PV and underlying storage are automatically deleted |
| `Recycle` | Deprecated — data wiped, PV made available again |

**Use `Retain` for production databases** — accidentally deleting a PVC should never auto-delete your data.

---

## 4. PersistentVolumeClaim (PVC) — Requesting Storage

### WHAT
A **PersistentVolumeClaim** is a request for storage by a user or application. It specifies how much storage is needed, what access mode is required, and optionally what StorageClass to use. Kubernetes finds a matching PV and **binds** them together.

### WHY
PVCs decouple the developer from storage infrastructure:
- Developer says: "I need 20Gi of ReadWriteOnce storage"
- Cluster figures out: "I have an EBS volume that matches — binding it"
- Developer doesn't care if it's EBS, NFS, or a local SSD

### HOW — PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: default        # ← PVCs are namespace-scoped
spec:
  accessModes:
    - ReadWriteOnce         # must match a PV's access modes

  resources:
    requests:
      storage: 20Gi         # minimum storage needed

  storageClassName: standard  # which StorageClass to use
                              # omit to use cluster default
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc

# Output:
# NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# postgres-pvc   Bound    postgres-pv   20Gi       RWO            standard       30s
#                ^^^^^^
#                STATUS: Pending → Bound (when matched with a PV)
```

**PVC Status:**
- `Pending` — no matching PV found yet (waiting for manual PV creation or dynamic provisioning)
- `Bound` — matched with a PV, ready to use
- `Lost` — previously bound PV is gone (data may be lost)

---

## 5. Using a PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
spec:
  containers:
    - name: postgres
      image: postgres:15-alpine
      env:
        - name: POSTGRES_PASSWORD
          value: "secret"
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata  # prevent data dir issue
      volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data     # where postgres stores data

  volumes:
    - name: postgres-storage
      persistentVolumeClaim:
        claimName: postgres-pvc    # ← reference the PVC by name
```

**Complete flow:**
```
Pod spec references PVC "postgres-pvc"
        │
        ▼
PVC "postgres-pvc" is bound to PV "postgres-pv"
        │
        ▼
PV "postgres-pv" is backed by AWS EBS vol-0a1b2c3d4e5f6789
        │
        ▼
kubelet mounts the EBS volume at /var/lib/postgresql/data
        │
        ▼
Postgres writes data → survives Pod restarts, rescheduling
```

---

## 6. StorageClass — Dynamic Provisioning

### WHAT
A **StorageClass** defines a "class" of storage and a **provisioner** that can automatically create PVs on demand. When a PVC is created with a StorageClass, Kubernetes automatically provisions the underlying storage (creates an EBS volume, etc.) and creates a PV for it — no manual PV creation needed.

### WHY
Manual PV creation is impractical at scale:
- 50 microservices each need storage → admin creates 50 PVs manually? No.
- Developer shouldn't need to know about EBS volume IDs
- Storage should be provisioned on demand, when requested

Dynamic provisioning via StorageClasses solves this:
- Developer creates a PVC → StorageClass automatically creates PV + underlying storage
- Developer has their storage in seconds

### HOW — StorageClass YAML

**Built-in StorageClasses in cloud providers:**
```bash
# See available StorageClasses in your cluster
kubectl get storageclasses
kubectl get sc    # shorthand

# In Minikube:
# NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   k8s.io/minikube-hostpath   Delete          Immediate

# In AWS EKS:
# NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer
# gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer
```

**Custom StorageClass for AWS EBS gp3:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com          # AWS EBS CSI driver
parameters:
  type: gp3                            # EBS volume type
  iopsPerGB: "10"                      # IOPS
  encrypted: "true"                    # encrypt at rest
  kmsKeyId: arn:aws:kms:...           # KMS key for encryption
reclaimPolicy: Retain                  # keep data when PVC deleted
allowVolumeExpansion: true             # allow resizing volumes
volumeBindingMode: WaitForFirstConsumer # provision only when Pod is scheduled
```

**StorageClass for different environments:**
```yaml
# Development: slow, cheap, delete on cleanup
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dev-storage
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete

---
# Production: fast, encrypted, retain data
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Dynamic Provisioning Flow

```
Developer creates PVC:
  storageClassName: fast-ssd
  storage: 20Gi
        │
        ▼
StorageClass "fast-ssd" provisioner triggers
        │
        ▼
AWS EBS CSI driver creates EBS gp3 volume (20Gi, encrypted)
        │
        ▼
Kubernetes creates PV automatically bound to this PVC
        │
        ▼
PVC status: Pending → Bound
        │
        ▼
Pod can now mount the PVC
```

---

## 7. Volume Binding Modes

| Mode | Behaviour |
|---|---|
| `Immediate` | PVC binds to a PV as soon as it's created, regardless of where the Pod will be scheduled |
| `WaitForFirstConsumer` | PVC binding is delayed until a Pod using this PVC is scheduled — then provisions in the same AZ as the Pod |

**Why `WaitForFirstConsumer` matters for EBS:**
EBS volumes are zone-specific (us-east-1a, us-east-1b, etc.). If PVC binds immediately in us-east-1a but the Pod gets scheduled to a node in us-east-1b — the Pod can't mount the volume! `WaitForFirstConsumer` prevents this by waiting to see which node/AZ the Pod lands on before provisioning.

---

## 8. Volume Expansion — Resizing Storage

### WHAT
If `allowVolumeExpansion: true` in the StorageClass, you can resize a PVC by editing its storage request.

### HOW
```bash
# Edit the PVC to request more storage
kubectl edit pvc postgres-pvc

# Change: storage: 20Gi → storage: 50Gi

# For filesystem-based volumes (ext4/xfs), the filesystem must be expanded too
# This happens automatically when:
# - The Pod is restarted (for offline resize)
# - Or automatically online (for supported CSI drivers)

# Check resize status
kubectl describe pvc postgres-pvc
# Look for: "Conditions: FileSystemResizePending: False" → done
```

---

## 9. Complete Production Example — PostgreSQL with Persistent Storage

This is a production-quality setup for running PostgreSQL in Kubernetes:

```yaml
# ── StorageClass (one-time cluster setup) ────────────────────────────────────
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Retain                   # NEVER auto-delete DB data
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# ── PersistentVolumeClaim ─────────────────────────────────────────────────────
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: postgres-storage
  resources:
    requests:
      storage: 50Gi

---
# ── Secret for DB credentials ─────────────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: production
type: Opaque
stringData:
  POSTGRES_USER: taskuser
  POSTGRES_PASSWORD: supersecretpassword
  POSTGRES_DB: taskdb

---
# ── PostgreSQL Deployment ─────────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: production
spec:
  replicas: 1            # single replica — use StatefulSet for HA (Day 22)
  selector:
    matchLabels:
      app: postgres
  strategy:
    type: Recreate       # ← IMPORTANT: use Recreate for databases
                         # RWO volumes can only be mounted by one node
                         # RollingUpdate would try to run 2 pods simultaneously
                         # causing mount conflicts
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          envFrom:
            - secretRef:
                name: postgres-secret
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3

      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-data-pvc     # ← mount the PVC

---
# ── Service ───────────────────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: production
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

**Why `strategy: Recreate` for databases with RWO volumes:**
```
RollingUpdate with RWO volume would try:
  Step 1: Start new postgres Pod → tries to mount the EBS volume
  But old postgres Pod still has it mounted!
  → EBS volumes can only be mounted by ONE node
  → New Pod stuck in "Pending" forever waiting for volume to be freed

Recreate avoids this:
  Step 1: Terminate old Pod → volume unmounted
  Step 2: Start new Pod → volume available → mounts successfully
  Brief downtime, but no stuck deployment
```

---

## 10. emptyDir — Temporary Pod-level Storage

### WHAT
`emptyDir` creates a temporary directory that exists as long as the Pod exists. All containers in the Pod can share it. When the Pod is deleted, the data is gone.

### WHY
Use for:
- Sharing data between containers in the same Pod (sidecar pattern)
- Cache/scratch space that doesn't need to survive Pod restarts
- Temporary computations

```yaml
spec:
  containers:
    - name: api
      volumeMounts:
        - name: shared-data
          mountPath: /app/cache
    - name: cache-warmer
      volumeMounts:
        - name: shared-data
          mountPath: /data

  volumes:
    - name: shared-data
      emptyDir: {}                   # basic — uses node disk
    - name: fast-cache
      emptyDir:
        medium: Memory               # tmpfs — uses RAM, faster but limited
        sizeLimit: 512Mi             # cap RAM usage
```

---

## 11. hostPath — Mount Node's Filesystem (Dev Only)

### WHAT
`hostPath` mounts a file or directory from the **host node's filesystem** into the Pod.

### WHY
Useful in development (Minikube) or for DaemonSet use cases (collecting node logs). **Never use in production for application data** — Pod could be rescheduled to a different node and lose data.

```yaml
volumes:
  - name: node-logs
    hostPath:
      path: /var/log              # path on the NODE's filesystem
      type: Directory             # Directory | File | DirectoryOrCreate | etc.
```

---

## 12. Volume Types Reference

| Volume Type | Persists beyond Pod? | Multi-node? | Use case |
|---|---|---|---|
| `emptyDir` | ❌ Pod lifetime | ❌ | Sidecar file sharing, scratch space |
| `hostPath` | ✅ (node-local) | ❌ | Dev only, DaemonSets |
| `configMap` | ✅ (in K8s) | ✅ | Config file injection (Day 16) |
| `secret` | ✅ (in K8s) | ✅ | Secret file injection (Day 16) |
| `persistentVolumeClaim` | ✅ | Depends on PV | Production storage |
| `awsElasticBlockStore` | ✅ | ❌ (RWO) | AWS block storage (use CSI driver) |
| `nfs` | ✅ | ✅ (RWX) | Shared file storage |
| `projected` | — | — | Combine multiple volume sources into one |
| `csi` | ✅ | Depends | Any CSI-compliant storage |

---

## 13. CSI — Container Storage Interface (Modern Storage)

### WHAT
**CSI (Container Storage Interface)** is a standard API that lets storage vendors write one driver that works with any container orchestrator (Kubernetes, Mesos, etc.).

### WHY
Before CSI, every storage vendor had to be hardcoded into Kubernetes source code. With CSI, vendors ship their driver as a DaemonSet — you install it in your cluster and get full support for that storage system.

### Common CSI drivers

| Driver | Storage system |
|---|---|
| `ebs.csi.aws.com` | AWS EBS |
| `efs.csi.aws.com` | AWS EFS (NFS-compatible, RWX) |
| `disk.csi.azure.com` | Azure Disk |
| `file.csi.azure.com` | Azure Files |
| `pd.csi.storage.gke.io` | GCP Persistent Disk |
| `nfs.csi.k8s.io` | Generic NFS |

---

## 14. PV Lifecycle — Complete State Machine

```
PV State Transitions:

Available → Bound → Released → (Retain: Available for manual reclaim)
                             → (Delete: PV and storage deleted)

Available:   PV exists and is not bound to any PVC
Bound:       PV is bound to a PVC — exclusively used
Released:    PVC that bound this PV was deleted
             PV still exists, data still there
             But NOT automatically available for new PVCs
             (admin must manually reclaim or reset)
Failed:      Automatic reclamation failed
```

```bash
# Check PV status
kubectl get pv

# NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
# postgres-pv   20Gi       RWO            Retain           Bound    default/postgres-pvc
# old-pv        20Gi       RWO            Retain           Released default/old-pvc (deleted)
```

---

## 15. Debugging Storage Problems

```bash
# ── PVC stuck in Pending ──────────────────────────────────────────────────────
kubectl describe pvc postgres-pvc
# Look in Events:
# "no persistent volumes available for this claim"
#   → No PV matches (size, access mode, storageclass mismatch)
#   → If using dynamic provisioning: check StorageClass provisioner is installed
# "waiting for first consumer to be scheduled before binding"
#   → WaitForFirstConsumer mode — normal until Pod is scheduled

# ── Pod stuck in Pending: "pod has unbound PersistentVolumeClaims" ────────────
kubectl describe pod postgres-pod
# → PVC not in Bound status yet — wait or fix PVC first

# ── Pod stuck: can't mount volume ─────────────────────────────────────────────
kubectl describe pod postgres-pod
# Events:
# "Unable to attach or mount volumes"
# "volume is already exclusively attached to one node"
#   → Old pod still running / old volume not released
#   → Use strategy: Recreate or wait for old pod to fully terminate

# ── Check what's using a PVC ──────────────────────────────────────────────────
kubectl get pods -o json | \
  jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="postgres-pvc") | .metadata.name'

# ── Manually resize a PVC (if StorageClass allows) ────────────────────────────
kubectl patch pvc postgres-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

---

## 16. Storage Summary — The Full Picture

```
For Minikube/dev:
  PVC (storageClassName: standard) → StorageClass → hostPath PV (auto-created)

For AWS EKS production:
  PVC (storageClassName: gp3) → StorageClass (ebs.csi.aws.com) → EBS gp3 volume (auto-created)
  
For shared file storage (AWS):
  PVC (storageClassName: efs) → StorageClass (efs.csi.aws.com) → EFS mount target (RWX!)

For on-premises:
  Admin manually creates PV pointing to NFS share
  Developer creates PVC → bound to PV
```

---

## 17. Hands-On Practice Tasks

```bash
# 1. Check available StorageClasses in Minikube
kubectl get storageclass

# 2. Create a PVC (Minikube will auto-provision a PV)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
EOF

# 3. Watch PVC become Bound
kubectl get pvc -w

# 4. Check the auto-created PV
kubectl get pv

# 5. Mount it in a Pod and write data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-test
spec:
  containers:
    - name: writer
      image: busybox
      command: ["/bin/sh", "-c", "echo 'persistent data' > /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: test-storage
          mountPath: /data
  volumes:
    - name: test-storage
      persistentVolumeClaim:
        claimName: test-pvc
EOF

# 6. Verify data was written
kubectl exec storage-test -- cat /data/test.txt

# 7. Delete the Pod and recreate — verify data persists
kubectl delete pod storage-test
kubectl apply -f pod.yaml  # same pod spec
kubectl exec storage-test -- cat /data/test.txt   # data still there!

# 8. Deploy the full PostgreSQL stack from Section 9
# 9. Connect to it and create a table, insert data
# 10. Delete and recreate the postgres Pod — verify data survives
kubectl delete pod -l app=postgres
kubectl get pods -w   # watch it come back
kubectl exec -it <new-pod> -- psql -U taskuser -d taskdb -c "SELECT * FROM your_table;"
```

---

## 18. Quiz — Test Yourself / Test Others

1. What are the three Kubernetes storage objects and how do they relate to each other?
2. What is a PersistentVolume and who typically creates it?
3. What is a PersistentVolumeClaim and who creates it?
4. What is the difference between `ReadWriteOnce` and `ReadWriteMany`?
5. What does a StorageClass do and why does it matter?
6. What is `volumeBindingMode: WaitForFirstConsumer` and why is it important for AWS EBS?
7. Why must you use `strategy: Recreate` (not RollingUpdate) for a Deployment that uses an RWO PVC?
8. What is the difference between `emptyDir` and a PVC?
9. What does a `Retain` reclaim policy mean? When should you use it?
10. A PVC has been in `Pending` status for 5 minutes. What are the possible causes and how would you debug it?

<details>
<summary>Answers</summary>

1. StorageClass = recipe/template for provisioning storage dynamically. PersistentVolume (PV) = the actual storage resource (EBS volume, NFS share, etc.) — cluster-scoped. PersistentVolumeClaim (PVC) = a request for storage from a namespace — Kubernetes binds it to a matching PV. Pods mount PVCs as volumes.

2. A PersistentVolume represents a real piece of storage in the cluster. With static provisioning, a cluster admin creates it manually (specifying the EBS volume ID, NFS path, etc.). With dynamic provisioning via StorageClasses, Kubernetes creates PVs automatically in response to PVC requests — no manual admin step needed.

3. A PersistentVolumeClaim is a request for storage by a developer or application. It specifies needed capacity, access mode, and StorageClass. Kubernetes finds (or dynamically provisions) a matching PV and binds them. PVCs are namespace-scoped, so different teams/namespaces have isolated claims.

4. ReadWriteOnce (RWO) means only one node can mount the volume for reading and writing at a time. ReadWriteMany (RWX) means multiple nodes can mount simultaneously for read-write. RWO is sufficient for most databases (one primary). RWX is needed for shared file storage accessed by Pods across multiple nodes (requires NFS, EFS, Azure Files, etc.).

5. A StorageClass defines a type of storage and a provisioner plugin that can create it on demand. When a PVC references a StorageClass, the provisioner automatically creates the underlying storage (e.g., an EBS volume) and a PV without any manual admin action. It enables self-service storage provisioning at scale.

6. WaitForFirstConsumer delays PVC binding until a Pod using the PVC is actually scheduled to a node. Without it, the PVC binds immediately — possibly provisioning an EBS volume in us-east-1a while the Pod ends up scheduled on a node in us-east-1b. Since EBS volumes are zone-specific, the Pod can't mount the volume. WaitForFirstConsumer ensures the volume is created in the same AZ as the node.

7. RWO (ReadWriteOnce) volumes can only be mounted by one node at a time. RollingUpdate creates a new Pod before terminating the old one. The new Pod tries to mount the EBS volume, but the old Pod still has it mounted on its node — resulting in the new Pod stuck in Pending indefinitely. Recreate terminates the old Pod first, freeing the volume, then starts the new Pod.

8. `emptyDir` is temporary storage that lives only as long as the Pod — created when the Pod starts, deleted when the Pod stops. It's useful for sharing data between containers in the same Pod. A PVC is persistent storage that outlives the Pod — the data remains even after the Pod is deleted, and new Pods can mount the same PVC.

9. Retain means when the PVC is deleted, the PV (and its underlying storage) is NOT automatically deleted — the data is preserved. The PV moves to Released state. An admin must manually decide to reclaim, repurpose, or delete it. Use Retain for production databases where accidental PVC deletion must never trigger data loss.

10. Causes: (1) No matching PV available — size, access mode, or storageClassName doesn't match any available PV. (2) StorageClass provisioner is not installed or misconfigured. (3) `WaitForFirstConsumer` mode — PVC waits until a Pod using it is scheduled. (4) Insufficient cloud quotas. Debug: `kubectl describe pvc <name>` — check the Events section for specific error messages. Also check `kubectl get storageclass` and `kubectl get pv` to see available options.

</details>

---

## 19. Summary

After today you can explain and implement:
- Why Pods need external storage — ephemeral containers, multi-node rescheduling
- All three storage objects: **StorageClass** (recipe), **PV** (actual storage), **PVC** (request)
- PV access modes: **RWO**, **ROX**, **RWX**, **RWOP** — and what each means
- PV reclaim policies: **Retain** (use for prod DBs), **Delete** (use for ephemeral data)
- **Dynamic provisioning** — PVC + StorageClass → automatic PV creation → no manual admin step
- **VolumeBindingMode: WaitForFirstConsumer** — critical for AZ-specific storage (EBS)
- Why databases need `strategy: Recreate` not `RollingUpdate` with RWO volumes
- `emptyDir` — temporary Pod-level shared storage
- `hostPath` — node filesystem mount (dev only)
- **CSI drivers** — modern, pluggable storage interface
- Full production PostgreSQL deployment with PVC, Secret, and Recreate strategy
- Storage debugging toolkit

**Coming up on Day 18:** Namespaces and Resource Management — resource quotas, LimitRanges, namespace isolation, and how to properly structure a production cluster for multiple teams and environments.
