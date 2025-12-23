# CKA Storage Scenarios (10% of Exam)

> **Exam Weight:** 10%  
> **Study Time:** ~11 hours across Days 2-3  
> **Total Scenarios:** 18

---

## Table of Contents
1. [Storage Concepts Overview](#storage-concepts-overview)
2. [Scenarios 1-6: PersistentVolumes and PersistentVolumeClaims](#scenarios-1-6-pv-and-pvc-basics)
3. [Scenarios 7-12: StorageClasses and Dynamic Provisioning](#scenarios-7-12-storageclasses-and-dynamic-provisioning)
4. [Scenarios 13-18: Volume Types and Advanced Configurations](#scenarios-13-18-volume-types-and-advanced-configurations)

---

## Storage Concepts Overview

### Theory (Read Before Practice)

#### Volume Types
| Volume Type | Persistence | Use Case |
|-------------|-------------|----------|
| `emptyDir` | Pod lifetime | Temp storage, cache, inter-container sharing |
| `hostPath` | Node lifetime | Development, single-node testing |
| `PersistentVolume` | Independent | Production workloads |
| `configMap/secret` | ConfigMap lifetime | Configuration data |

#### Access Modes
| Mode | Abbreviation | Description |
|------|--------------|-------------|
| ReadWriteOnce | RWO | Single node can mount read-write |
| ReadOnlyMany | ROX | Many nodes can mount read-only |
| ReadWriteMany | RWX | Many nodes can mount read-write |
| ReadWriteOncePod | RWOP | Single pod can mount read-write |

#### Reclaim Policies
| Policy | Behavior |
|--------|----------|
| Retain | PV data preserved after PVC deletion (manual cleanup) |
| Delete | PV and underlying storage deleted with PVC |
| Recycle | Deprecated - data scrubbed and PV made available |

#### PV/PVC Binding Process
```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  PersistentVolume │────▶│     Binding      │◀────│ PersistentVolume │
│      (PV)        │     │                  │     │    Claim (PVC)   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
       │                                                  │
       │ Capacity: 10Gi                    Requested: 5Gi │
       │ AccessMode: RWO                   AccessMode: RWO│
       │ StorageClass: manual              StorageClass: manual
       └──────────────────────────────────────────────────┘
                        Match & Bind
```

---

## Scenarios 1-6: PV and PVC Basics

---

### Scenario 1: Create a PersistentVolume with hostPath

**Objective:** Create a PersistentVolume using hostPath storage.

**Requirements:**
- Name: `pv-log`
- Capacity: 100Mi
- Access Mode: ReadWriteMany
- hostPath: `/pv/log`
- Reclaim Policy: Retain

**Your Task:**
```bash
# Create the directory on all nodes first
sudo mkdir -p /pv/log

# Create the PersistentVolume
```

<details>
<summary>💡 Solution</summary>

```yaml
# pv-log.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /pv/log
```

```bash
kubectl apply -f pv-log.yaml

# Verify
kubectl get pv pv-log
kubectl describe pv pv-log
```

**Expected Output:**
```
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-log   100Mi      RWX            Retain           Available                          5s
```

</details>

---

### Scenario 2: Create a PersistentVolumeClaim

**Objective:** Create a PVC that binds to the PV created in Scenario 1.

**Requirements:**
- Name: `claim-log-1`
- Requested Storage: 50Mi
- Access Mode: ReadWriteMany

**Your Task:**
```bash
# Create a PVC that will bind to pv-log
```

<details>
<summary>💡 Solution</summary>

```yaml
# claim-log-1.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Mi
```

```bash
kubectl apply -f claim-log-1.yaml

# Verify binding
kubectl get pvc claim-log-1
kubectl get pv pv-log
```

**Expected Output:**
```
NAME          STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim-log-1   Bound    pv-log   100Mi      RWX                           5s
```

**Note:** PVC requested 50Mi but got 100Mi because PV capacity is 100Mi (smallest matching PV).

</details>

---

### Scenario 3: Mount PVC in a Pod

**Objective:** Create a Pod that uses the PVC from Scenario 2.

**Requirements:**
- Pod name: `webapp`
- Image: `nginx`
- Mount the PVC at `/var/log/nginx`

**Your Task:**
```bash
# Create a pod that mounts claim-log-1
```

<details>
<summary>💡 Solution</summary>

```yaml
# webapp.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
  volumes:
  - name: log-volume
    persistentVolumeClaim:
      claimName: claim-log-1
```

```bash
kubectl apply -f webapp.yaml

# Verify
kubectl get pod webapp
kubectl describe pod webapp | grep -A5 Volumes
```

**Verification:**
```bash
# Write data to the volume
kubectl exec webapp -- sh -c 'echo "test log" > /var/log/nginx/test.log'

# Verify data
kubectl exec webapp -- cat /var/log/nginx/test.log
```

</details>

---

### Scenario 4: PVC in a Specific Namespace

**Objective:** Create a PVC in a non-default namespace.

**Requirements:**
- Namespace: `finance`
- PVC name: `finance-data`
- Requested Storage: 500Mi
- Access Mode: ReadWriteOnce

**Your Task:**
```bash
# Create the namespace and PVC
```

<details>
<summary>💡 Solution</summary>

```bash
# Create namespace
kubectl create namespace finance

# Create PV for the namespace (PVs are cluster-scoped)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-finance
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /pv/finance
EOF

# Create PVC in finance namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: finance-data
  namespace: finance
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF

# Verify
kubectl get pvc -n finance
```

**Key Point:** PersistentVolumes are cluster-scoped, but PersistentVolumeClaims are namespace-scoped.

</details>

---

### Scenario 5: Expand a PersistentVolumeClaim

**Objective:** Resize an existing PVC to increase its storage.

**Requirements:**
- Create a StorageClass that allows expansion
- Create a 1Gi PVC
- Expand it to 2Gi

**Your Task:**
```bash
# Create an expandable StorageClass and PVC, then expand it
```

<details>
<summary>💡 Solution</summary>

```yaml
# expandable-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# Create a PV for this StorageClass
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-expandable
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: expandable
  hostPath:
    path: /pv/expandable
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: expandable
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f expandable-sc.yaml

# Create a pod to trigger binding
kubectl run test-pod --image=nginx --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"expandable-pvc"}}]}}'

# Expand the PVC
kubectl patch pvc expandable-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

# Verify
kubectl get pvc expandable-pvc
```

**Note:** For file-system based volumes, expansion may require pod restart.

</details>

---

### Scenario 6: Delete PVC and Observe Reclaim Policy

**Objective:** Understand what happens to a PV when its PVC is deleted.

**Requirements:**
- Create a PV with `Retain` policy
- Create a PVC that binds to it
- Delete the PVC and observe PV state
- Make the PV available again

**Your Task:**
```bash
# Create PV with Retain, bind PVC, delete PVC, then reclaim PV
```

<details>
<summary>💡 Solution</summary>

```bash
# Create PV with Retain policy
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-retain-demo
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /pv/retain-demo
EOF

# Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-retain-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
EOF

# Check binding
kubectl get pv,pvc

# Delete PVC
kubectl delete pvc pvc-retain-demo

# Check PV status - it should be "Released", not "Available"
kubectl get pv pv-retain-demo
```

**Output after PVC deletion:**
```
NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                     STORAGECLASS
pv-retain-demo    100Mi      RWO            Retain           Released   default/pvc-retain-demo   
```

**To make PV Available again:**
```bash
# Remove the claimRef to make it Available
kubectl patch pv pv-retain-demo --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'

# Verify
kubectl get pv pv-retain-demo
# Status should now be "Available"
```

</details>

---

## Scenarios 7-12: StorageClasses and Dynamic Provisioning

---

### Scenario 7: Create a StorageClass

**Objective:** Create a custom StorageClass for local storage.

**Requirements:**
- Name: `local-storage`
- Provisioner: `kubernetes.io/no-provisioner`
- Volume Binding Mode: `WaitForFirstConsumer`

**Your Task:**
```bash
# Create the StorageClass
```

<details>
<summary>💡 Solution</summary>

```yaml
# local-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

```bash
kubectl apply -f local-storage-class.yaml

# Verify
kubectl get storageclass local-storage
kubectl describe storageclass local-storage
```

**Volume Binding Modes:**
- `Immediate`: Bind as soon as PVC is created
- `WaitForFirstConsumer`: Delay binding until Pod using PVC is scheduled (better for topology-aware provisioning)

</details>

---

### Scenario 8: Use StorageClass in PVC

**Objective:** Create a PVC that uses a specific StorageClass.

**Requirements:**
- Create a PV with storageClassName: `local-storage`
- Create a PVC that requests `local-storage` class
- Verify binding

**Your Task:**
```bash
# Create PV and PVC with StorageClass
```

<details>
<summary>💡 Solution</summary>

```yaml
# pv-with-class.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 200Mi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  hostPath:
    path: /pv/local
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 200Mi
```

```bash
kubectl apply -f pv-with-class.yaml

# With WaitForFirstConsumer, PVC stays Pending until pod is created
kubectl get pvc local-pvc
# STATUS: Pending

# Create a pod to trigger binding
kubectl run local-pod --image=nginx --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"local-pvc"}}]}}'

# Now check PVC
kubectl get pvc local-pvc
# STATUS: Bound
```

</details>

---

### Scenario 9: Set Default StorageClass

**Objective:** Make a StorageClass the default for the cluster.

**Requirements:**
- Create a StorageClass `standard`
- Mark it as default
- Verify PVC without storageClassName uses it

**Your Task:**
```bash
# Create a default StorageClass
```

<details>
<summary>💡 Solution</summary>

```yaml
# default-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f default-sc.yaml

# Verify it's marked as default
kubectl get storageclass
# NAME                 PROVISIONER                    AGE
# standard (default)   kubernetes.io/no-provisioner   5s

# Create PVC without storageClassName
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
EOF

# Check the PVC - it should have standard as storageClassName
kubectl get pvc default-test-pvc -o jsonpath='{.spec.storageClassName}'
# Output: standard
```

**To remove default:**
```bash
kubectl patch storageclass standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

</details>

---

### Scenario 10: StorageClass with Reclaim Policy

**Objective:** Create StorageClasses with different reclaim policies.

**Requirements:**
- Create `sc-retain` with Retain policy
- Create `sc-delete` with Delete policy
- Understand when to use each

**Your Task:**
```bash
# Create two StorageClasses with different reclaim policies
```

<details>
<summary>💡 Solution</summary>

```yaml
# storage-classes-reclaim.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-retain
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain  # Data preserved after PVC deletion
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-delete
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete  # Data deleted with PVC
```

```bash
kubectl apply -f storage-classes-reclaim.yaml
kubectl get storageclass
```

**When to use:**
- **Retain**: Production data, databases, anything you can't afford to lose
- **Delete**: Development environments, temporary data, cost optimization

</details>

---

### Scenario 11: PVC with Specific Selector

**Objective:** Create a PVC that selects a specific PV using labels.

**Requirements:**
- Create a PV with label `type: ssd`
- Create a PVC that selects PVs with `type: ssd`

**Your Task:**
```bash
# Create PV with label and PVC with selector
```

<details>
<summary>💡 Solution</summary>

```yaml
# pv-with-label.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ssd
  labels:
    type: ssd
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /pv/ssd
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hdd
  labels:
    type: hdd
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /pv/hdd
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-ssd-only
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  selector:
    matchLabels:
      type: ssd
```

```bash
kubectl apply -f pv-with-label.yaml

# Verify the PVC bound to the correct PV
kubectl get pvc pvc-ssd-only -o jsonpath='{.spec.volumeName}'
# Output: pv-ssd (NOT pv-hdd)
```

</details>

---

### Scenario 12: Troubleshoot PVC Pending State

**Objective:** Debug why a PVC is stuck in Pending state.

**Requirements:**
- Identify common reasons for PVC Pending
- Fix the issue and get PVC bound

**Setup (run this first):**
```bash
# Create a problematic PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stuck-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: non-existent-class
  resources:
    requests:
      storage: 10Gi
EOF
```

**Your Task:**
```bash
# Debug why stuck-pvc is Pending and fix it
```

<details>
<summary>💡 Solution</summary>

```bash
# Check PVC status
kubectl get pvc stuck-pvc
# STATUS: Pending

# Describe to find the reason
kubectl describe pvc stuck-pvc
# Events:
#   Warning  ProvisioningFailed  ... storageclass.storage.k8s.io "non-existent-class" not found

# List available StorageClasses
kubectl get storageclass

# Option 1: Create the missing StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: non-existent-class
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

# Option 2: Delete and recreate PVC with existing StorageClass
kubectl delete pvc stuck-pvc
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stuck-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""  # Use empty string for no StorageClass
  resources:
    requests:
      storage: 10Gi
EOF
```

**Common reasons for PVC Pending:**
1. No matching PV (capacity, access mode, storage class)
2. StorageClass doesn't exist
3. No available PVs with matching labels (if selector used)
4. `WaitForFirstConsumer` - waiting for Pod

</details>

---

## Scenarios 13-18: Volume Types and Advanced Configurations

---

### Scenario 13: Use emptyDir Volume

**Objective:** Create a Pod with emptyDir volume for sharing data between containers.

**Requirements:**
- Pod name: `multi-container`
- Two containers: `writer` and `reader`
- Writer writes to shared volume
- Reader reads from shared volume

**Your Task:**
```bash
# Create a multi-container pod with emptyDir
```

<details>
<summary>💡 Solution</summary>

```yaml
# multi-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'while true; do date >> /shared/log.txt; sleep 5; done']
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: reader
    image: busybox
    command: ['sh', '-c', 'tail -f /shared/log.txt']
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

```bash
kubectl apply -f multi-container.yaml

# Verify
kubectl logs multi-container -c reader
```

**emptyDir options:**
```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory  # RAM-backed (tmpfs)
    sizeLimit: 100Mi
```

</details>

---

### Scenario 14: Use hostPath Volume (Different Types)

**Objective:** Understand different hostPath types.

**Requirements:**
- Create Pod with hostPath type `DirectoryOrCreate`
- Create Pod with hostPath type `FileOrCreate`

**Your Task:**
```bash
# Create pods with different hostPath types
```

<details>
<summary>💡 Solution</summary>

```yaml
# hostpath-types.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-dir
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'ls -la /test-dir && sleep 3600']
    volumeMounts:
    - name: dir-volume
      mountPath: /test-dir
  volumes:
  - name: dir-volume
    hostPath:
      path: /tmp/test-dir
      type: DirectoryOrCreate  # Creates directory if not exists
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-file
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'cat /test-file && sleep 3600']
    volumeMounts:
    - name: file-volume
      mountPath: /test-file
  volumes:
  - name: file-volume
    hostPath:
      path: /tmp/test-file
      type: FileOrCreate  # Creates file if not exists
```

**hostPath Types:**
| Type | Description |
|------|-------------|
| `""` | Empty string - no checks, default |
| `DirectoryOrCreate` | Creates directory if missing |
| `Directory` | Directory must exist |
| `FileOrCreate` | Creates empty file if missing |
| `File` | File must exist |
| `Socket` | Unix socket must exist |
| `CharDevice` | Character device must exist |
| `BlockDevice` | Block device must exist |

</details>

---

### Scenario 15: Mount ConfigMap as Volume

**Objective:** Mount a ConfigMap as a volume in a Pod.

**Requirements:**
- Create ConfigMap with application config
- Mount as volume at `/etc/config`
- Verify files are created

**Your Task:**
```bash
# Create ConfigMap and mount it
```

<details>
<summary>💡 Solution</summary>

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=database_host=mysql.default.svc \
  --from-literal=database_port=3306 \
  --from-literal=log_level=debug

# Create Pod that mounts ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-mount-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /etc/config && cat /etc/config/* && sleep 3600']
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
EOF

# Verify
kubectl logs config-mount-pod
```

**Output:**
```
-rw-r--r-- database_host
-rw-r--r-- database_port
-rw-r--r-- log_level
mysql.default.svc3306debug
```

</details>

---

### Scenario 16: Mount Secret as Volume

**Objective:** Mount a Secret as a volume with specific permissions.

**Requirements:**
- Create Secret with credentials
- Mount at `/etc/secrets` with mode 0400
- Only mount specific keys

**Your Task:**
```bash
# Create Secret and mount with restricted permissions
```

<details>
<summary>💡 Solution</summary>

```bash
# Create Secret
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --from-literal=api-key=abc123

# Create Pod with selective mounting
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-mount-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /etc/secrets && cat /etc/secrets/* && sleep 3600']
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-creds
      defaultMode: 0400  # Read-only for owner
      items:  # Only mount specific keys
      - key: username
        path: db-user
      - key: password
        path: db-pass
EOF

# Verify
kubectl exec secret-mount-pod -- ls -la /etc/secrets
# -r--------  db-pass
# -r--------  db-user

kubectl exec secret-mount-pod -- cat /etc/secrets/db-user
# admin
```

</details>

---

### Scenario 17: Use Projected Volumes

**Objective:** Combine multiple volume sources into a single mount.

**Requirements:**
- Create a projected volume with ConfigMap and Secret
- Mount both at `/etc/projected`

**Your Task:**
```bash
# Create projected volume with ConfigMap and Secret
```

<details>
<summary>💡 Solution</summary>

```bash
# Create ConfigMap
kubectl create configmap config-data --from-literal=config.properties="key=value"

# Create Secret
kubectl create secret generic secret-data --from-literal=credentials="user:pass"

# Create Pod with projected volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: projected-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /etc/projected && sleep 3600']
    volumeMounts:
    - name: all-in-one
      mountPath: /etc/projected
  volumes:
  - name: all-in-one
    projected:
      sources:
      - configMap:
          name: config-data
      - secret:
          name: secret-data
      - downwardAPI:
          items:
          - path: podname
            fieldRef:
              fieldPath: metadata.name
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
EOF

# Verify
kubectl exec projected-pod -- ls /etc/projected
# config.properties
# credentials
# podname
# token
```

</details>

---

### Scenario 18: Troubleshoot Volume Mount Issues

**Objective:** Debug and fix volume mounting problems.

**Setup (run this first):**
```bash
# Create a broken Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-volume-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: missing-pvc
EOF
```

**Your Task:**
```bash
# Debug why the pod is not running and fix it
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod broken-volume-pod
# STATUS: Pending

# Describe to find the issue
kubectl describe pod broken-volume-pod
# Events:
#   Warning  FailedScheduling  ... persistentvolumeclaim "missing-pvc" not found

# Create the missing PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fix-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /pv/fix
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: missing-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF

# Check pod again - might need to delete and recreate
kubectl delete pod broken-volume-pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-volume-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: missing-pvc
EOF

# Verify
kubectl get pod broken-volume-pod
# STATUS: Running
```

**Common Volume Issues:**
1. PVC not found
2. PVC in wrong namespace
3. PV capacity doesn't match PVC request
4. Access modes don't match
5. StorageClass mismatch
6. Node doesn't have access to storage (hostPath on wrong node)

</details>

---

## Quick Reference Commands

```bash
# PersistentVolume operations
kubectl get pv
kubectl describe pv <pv-name>
kubectl delete pv <pv-name>

# PersistentVolumeClaim operations
kubectl get pvc
kubectl get pvc -A  # All namespaces
kubectl describe pvc <pvc-name>
kubectl delete pvc <pvc-name>

# StorageClass operations
kubectl get storageclass
kubectl get sc  # Short form
kubectl describe sc <sc-name>

# Check what PVC a PV is bound to
kubectl get pv <pv-name> -o jsonpath='{.spec.claimRef.name}'

# Check what PV a PVC is bound to
kubectl get pvc <pvc-name> -o jsonpath='{.spec.volumeName}'

# Get all storage resources
kubectl get pv,pvc,sc

# Patch PV to make it Available again (after Released)
kubectl patch pv <pv-name> --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'
```

---

## Exam Tips for Storage

1. **Know access modes:** RWO is most common, RWX requires specific storage backends
2. **Reclaim policies matter:** Use Retain for important data
3. **StorageClass binding modes:** WaitForFirstConsumer is topology-aware
4. **PVC stays Pending:** Check events with `kubectl describe pvc`
5. **Namespace awareness:** PVs are cluster-scoped, PVCs are namespaced
6. **Quick YAML generation:** Use `kubectl explain pv.spec` for field reference

---

**Next:** Move to [02-TROUBLESHOOTING-SCENARIOS.md](./02-TROUBLESHOOTING-SCENARIOS.md)
