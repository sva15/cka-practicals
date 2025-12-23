# CKA Cluster Admin Scenarios Part 1 (25% of Exam)

> **Exam Weight:** 25%  
> **Study Time:** ~28 hours across Days 1-3  
> **This File:** RBAC, kubeadm, etcd (16 scenarios)  
> **See Also:** [04B-CLUSTER-ADMIN-SCENARIOS-2.md](./04B-CLUSTER-ADMIN-SCENARIOS-2.md) for Helm, Kustomize, CRDs

---

## Table of Contents
1. [RBAC Scenarios (1-8)](#rbac-scenarios)
2. [kubeadm Scenarios (9-12)](#kubeadm-scenarios)
3. [etcd Scenarios (13-16)](#etcd-scenarios)

---

## RBAC Scenarios

### Scenario 1: Create a ServiceAccount

**Objective:** Create and use a ServiceAccount.

**Requirements:**
- Namespace: `dev`
- ServiceAccount: `dev-sa`

**Your Task:**
```bash
# Create namespace and ServiceAccount
```

<details>
<summary>💡 Solution</summary>

```bash
# Create namespace
kubectl create namespace dev

# Create ServiceAccount
kubectl create serviceaccount dev-sa -n dev

# Verify
kubectl get serviceaccount dev-sa -n dev
kubectl describe sa dev-sa -n dev

# Use in a pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dev-pod
  namespace: dev
spec:
  serviceAccountName: dev-sa
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
EOF

# Verify pod uses the SA
kubectl get pod dev-pod -n dev -o jsonpath='{.spec.serviceAccountName}'
```

</details>

---

### Scenario 2: Create a Role

**Objective:** Create a Role with specific permissions.

**Requirements:**
- Namespace: `dev`
- Role: `pod-reader`
- Permissions: get, list, watch pods

**Your Task:**
```bash
# Create a Role
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative method
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

# Declarative method
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

# Verify
kubectl describe role pod-reader -n dev
```

</details>

---

### Scenario 3: Create a RoleBinding

**Objective:** Bind Role to ServiceAccount.

**Requirements:**
- Bind `pod-reader` role to `dev-sa`
- Namespace: `dev`

**Your Task:**
```bash
# Create RoleBinding
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative
kubectl create rolebinding dev-pod-reader \
  --role=pod-reader \
  --serviceaccount=dev:dev-sa \
  -n dev

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-reader
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify
kubectl describe rolebinding dev-pod-reader -n dev

# Test permissions
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:dev-sa
# yes

kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:dev-sa
# no
```

</details>

---

### Scenario 4: Create a ClusterRole

**Objective:** Create cluster-wide permissions.

**Requirements:**
- ClusterRole: `node-reader`
- Permissions: get, list, watch nodes

**Your Task:**
```bash
# Create ClusterRole
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
EOF

# Verify
kubectl describe clusterrole node-reader
```

</details>

---

### Scenario 5: Create a ClusterRoleBinding

**Objective:** Bind ClusterRole to ServiceAccount.

**Requirements:**
- Bind `node-reader` to `dev-sa` in `dev` namespace

**Your Task:**
```bash
# Create ClusterRoleBinding
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative
kubectl create clusterrolebinding dev-node-reader \
  --clusterrole=node-reader \
  --serviceaccount=dev:dev-sa

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-node-reader
subjects:
- kind: ServiceAccount
  name: dev-sa
  namespace: dev
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify - dev-sa can now read nodes from any namespace
kubectl auth can-i list nodes --as=system:serviceaccount:dev:dev-sa
# yes
```

</details>

---

### Scenario 6: Create Full Admin Role for Namespace

**Objective:** Grant full access to a namespace.

**Requirements:**
- User: `jane`
- Full access to namespace `production`

**Your Task:**
```bash
# Create admin access for jane in production namespace
```

<details>
<summary>💡 Solution</summary>

```bash
# Create namespace
kubectl create namespace production

# Option 1: Use built-in admin ClusterRole with RoleBinding
kubectl create rolebinding jane-admin \
  --clusterrole=admin \
  --user=jane \
  -n production

# Option 2: Create custom full-access Role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-namespace-admin
  namespace: production
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify
kubectl auth can-i "*" "*" -n production --as=jane
# yes
```

</details>

---

### Scenario 7: Debug RBAC Issues

**Objective:** Troubleshoot permission denied errors.

**Setup:**
```bash
kubectl create sa limited-sa -n default
kubectl run test-pod --image=busybox --serviceaccount=limited-sa -- sleep 3600
```

**Your Task:**
```bash
# User reports: "Error from server (Forbidden): pods is forbidden"
# Diagnose and fix
```

<details>
<summary>💡 Solution</summary>

```bash
# Check what SA can do
kubectl auth can-i --list --as=system:serviceaccount:default:limited-sa

# Check for RoleBindings
kubectl get rolebindings -A | grep limited-sa
kubectl get clusterrolebindings | grep limited-sa

# No bindings! Create appropriate access
kubectl create role pod-manager \
  --verb=get,list,create,delete \
  --resource=pods

kubectl create rolebinding limited-sa-pod-manager \
  --role=pod-manager \
  --serviceaccount=default:limited-sa

# Verify
kubectl auth can-i list pods --as=system:serviceaccount:default:limited-sa
# yes

# Test from inside pod
kubectl exec test-pod -- wget -qO- --header="Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods --no-check-certificate
```

</details>

---

### Scenario 8: Create Read-Only Cluster Access

**Objective:** Create cluster-wide read-only access.

**Requirements:**
- ServiceAccount: `monitoring-sa`
- Can read all resources cluster-wide

**Your Task:**
```bash
# Create read-only cluster access
```

<details>
<summary>💡 Solution</summary>

```bash
# Create ServiceAccount
kubectl create sa monitoring-sa

# Use built-in view ClusterRole
kubectl create clusterrolebinding monitoring-view \
  --clusterrole=view \
  --serviceaccount=default:monitoring-sa

# Verify - can read, cannot modify
kubectl auth can-i list pods -A --as=system:serviceaccount:default:monitoring-sa
# yes

kubectl auth can-i delete pods -A --as=system:serviceaccount:default:monitoring-sa
# no

kubectl auth can-i list nodes --as=system:serviceaccount:default:monitoring-sa
# yes

# List all permissions
kubectl auth can-i --list --as=system:serviceaccount:default:monitoring-sa
```

</details>

---

## kubeadm Scenarios

### Scenario 9: Initialize a Cluster with kubeadm

**Objective:** Set up a new Kubernetes cluster.

**Your Task:**
```bash
# Initialize a new cluster
```

<details>
<summary>💡 Solution</summary>

```bash
# Prerequisites (on all nodes)
sudo swapoff -a
sudo modprobe br_netfilter
echo 1 | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables

# Initialize cluster (on master)
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP>

# Set up kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Join workers (command from kubeadm init output)
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>

# Verify
kubectl get nodes
```

</details>

---

### Scenario 10: Upgrade Cluster with kubeadm

**Objective:** Upgrade cluster from one version to another.

**Your Task:**
```bash
# Upgrade cluster from 1.28 to 1.29
```

<details>
<summary>💡 Solution</summary>

```bash
# === Control Plane Node ===

# Check current version
kubectl get nodes
kubeadm version

# Update package repos
sudo apt update
sudo apt-cache madison kubeadm | head -5

# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.29.0-1.1
sudo apt-mark hold kubeadm

# Verify
kubeadm version

# Plan upgrade
sudo kubeadm upgrade plan

# Apply upgrade
sudo kubeadm upgrade apply v1.29.0

# Drain node
kubectl drain <control-plane-node> --ignore-daemonsets

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon
kubectl uncordon <control-plane-node>

# === Worker Nodes (repeat for each) ===
# Drain
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# On worker:
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=1.29.0-1.1 kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
sudo apt-mark hold kubeadm kubelet kubectl

sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On master:
kubectl uncordon <worker-node>

# Verify
kubectl get nodes
```

</details>

---

### Scenario 11: Generate New Join Token

**Objective:** Create new token for adding workers.

**Your Task:**
```bash
# Generate join command for new worker
```

<details>
<summary>💡 Solution</summary>

```bash
# Simple method - print full join command
kubeadm token create --print-join-command

# Or step by step:

# List existing tokens
kubeadm token list

# Create new token (24h validity by default)
kubeadm token create

# Get CA cert hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'

# Or:
kubeadm token create --print-join-command

# Create token with longer TTL
kubeadm token create --ttl 48h --print-join-command

# Delete a token
kubeadm token delete <token>
```

</details>

---

### Scenario 12: Explore kubeadm Configuration

**Objective:** Understand kubeadm config and static pods.

**Your Task:**
```bash
# Examine kubeadm configuration
```

<details>
<summary>💡 Solution</summary>

```bash
# View kubeadm config in cluster
kubectl get cm -n kube-system kubeadm-config -o yaml

# Static pod manifests location
ls -la /etc/kubernetes/manifests/
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
# etcd.yaml

# View a static pod manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -50

# Certificate locations
ls -la /etc/kubernetes/pki/
# ca.crt, ca.key
# apiserver.crt, apiserver.key
# etcd/ca.crt, etcd/server.crt

# Check certificate expiration
kubeadm certs check-expiration

# Renew certificates
sudo kubeadm certs renew all

# View kubelet config
cat /var/lib/kubelet/config.yaml
```

</details>

---

## etcd Scenarios

### Scenario 13: Check etcd Health

**Objective:** Verify etcd cluster is healthy.

**Your Task:**
```bash
# Check etcd cluster health
```

<details>
<summary>💡 Solution</summary>

```bash
# Get etcd pod info
kubectl get pods -n kube-system | grep etcd

# Set up etcdctl alias (add to .bashrc for exam)
export ETCDCTL_API=3
alias etcdctl='etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

# Check endpoint health
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Member list
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Endpoint status (shows leader)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

</details>

---

### Scenario 14: Backup etcd

**Objective:** Create etcd snapshot backup.

**Your Task:**
```bash
# Take etcd backup
```

<details>
<summary>💡 Solution</summary>

```bash
# Create backup directory
sudo mkdir -p /backup

# Take snapshot
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
sudo ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table

# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 3e9bf... |    12345 |       1234 |     2.3 MB |
# +----------+----------+------------+------------+

# Verify file exists
ls -lh /backup/etcd-snapshot.db
```

**Exam Tip:** Know the cert paths by heart!
- CA: `/etc/kubernetes/pki/etcd/ca.crt`
- Cert: `/etc/kubernetes/pki/etcd/server.crt` 
- Key: `/etc/kubernetes/pki/etcd/server.key`

</details>

---

### Scenario 15: Restore etcd from Backup

**Objective:** Restore etcd cluster from snapshot.

**Your Task:**
```bash
# Restore etcd from backup
```

<details>
<summary>💡 Solution</summary>

```bash
# Step 1: Stop kube-apiserver (move manifest)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Step 2: Verify API server is down
kubectl get nodes  # Should fail

# Step 3: Restore snapshot to new directory
sudo ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# Step 4: Update etcd manifest to use new data directory
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' \
  /etc/kubernetes/manifests/etcd.yaml

# Also update the volume hostPath
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change:
#   - hostPath:
#       path: /var/lib/etcd
# To:
#   - hostPath:
#       path: /var/lib/etcd-restored

# Step 5: Restart etcd (kubelet will do it automatically)
# Or restart kubelet to force restart
sudo systemctl restart kubelet

# Step 6: Restore API server
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Step 7: Wait and verify
sleep 30
kubectl get nodes
kubectl get pods -A
```

</details>

---

### Scenario 16: Explore etcd Data

**Objective:** Query data stored in etcd.

**Your Task:**
```bash
# View data in etcd
```

<details>
<summary>💡 Solution</summary>

```bash
# List all keys
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20

# Get specific resource (e.g., a secret)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret

# List all namespaces stored
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/namespaces --prefix --keys-only

# Watch for changes (useful for debugging)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  watch /registry --prefix
```

</details>

---

## Quick Reference - RBAC

```bash
# ServiceAccount
kubectl create sa NAME [-n NAMESPACE]

# Role/ClusterRole
kubectl create role NAME --verb=V --resource=R [-n NAMESPACE]
kubectl create clusterrole NAME --verb=V --resource=R

# RoleBinding/ClusterRoleBinding
kubectl create rolebinding NAME --role=ROLE --serviceaccount=NS:SA [-n NAMESPACE]
kubectl create clusterrolebinding NAME --clusterrole=CR --serviceaccount=NS:SA

# Test permissions
kubectl auth can-i VERB RESOURCE [-n NAMESPACE] [--as=USER]
kubectl auth can-i --list [--as=USER]
```

## Quick Reference - etcd

```bash
# Always set API version
export ETCDCTL_API=3

# Common flags
--endpoints=https://127.0.0.1:2379
--cacert=/etc/kubernetes/pki/etcd/ca.crt
--cert=/etc/kubernetes/pki/etcd/server.crt
--key=/etc/kubernetes/pki/etcd/server.key

# Key commands
etcdctl snapshot save FILE
etcdctl snapshot status FILE
etcdctl snapshot restore FILE --data-dir=DIR
etcdctl endpoint health
etcdctl member list
```

---

**Next:** [04B-CLUSTER-ADMIN-SCENARIOS-2.md](./04B-CLUSTER-ADMIN-SCENARIOS-2.md) for Helm, Kustomize, and CRDs
