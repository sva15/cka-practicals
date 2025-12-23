# CKA Troubleshooting Scenarios (30% of Exam)

> **Exam Weight:** 30% (Highest!)  
> **Study Time:** ~34 hours across Days 5-7  
> **Total Scenarios:** 38

---

## Table of Contents
1. [Troubleshooting Methodology](#troubleshooting-methodology)
2. [Cluster & Node Troubleshooting (Scenarios 1-10)](#cluster--node-troubleshooting)
3. [Control Plane Troubleshooting (Scenarios 11-18)](#control-plane-troubleshooting)
4. [Application Troubleshooting (Scenarios 19-28)](#application-troubleshooting)
5. [Networking Troubleshooting (Scenarios 29-38)](#networking-troubleshooting)

---

## Troubleshooting Methodology

### The 5-Step Approach

```
┌─────────────────────────────────────────────────────────────┐
│  1. IDENTIFY → 2. GATHER → 3. ANALYZE → 4. FIX → 5. VERIFY │
└─────────────────────────────────────────────────────────────┘
```

| Step | Actions | Commands |
|------|---------|----------|
| 1. Identify | What's broken? Scope the issue | `kubectl get nodes`, `kubectl get pods -A` |
| 2. Gather | Collect logs, events, status | `kubectl describe`, `kubectl logs`, `journalctl` |
| 3. Analyze | Find root cause | Review error messages, check configs |
| 4. Fix | Apply the solution | Edit manifests, restart services |
| 5. Verify | Confirm resolution | Check status, test functionality |

### Essential Commands

```bash
# Cluster overview
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by='.lastTimestamp'

# Node details
kubectl describe node <node-name>
kubectl top node

# Pod debugging
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> [-c container] [--previous]
kubectl exec -it <pod> -n <ns> -- /bin/sh

# System services (on node)
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
sudo crictl ps
```

---

## Cluster & Node Troubleshooting

### Scenario 1: Node Not Ready

**Objective:** Diagnose and fix a node that shows NotReady status.

**Setup:** SSH to worker-1 and stop kubelet:
```bash
sudo systemctl stop kubelet
```

**Your Task:**
```bash
# From master node, diagnose why worker-1 is NotReady and fix it
```

<details>
<summary>💡 Solution</summary>

```bash
# Step 1: Identify
kubectl get nodes
# NAME       STATUS     ROLES           AGE   VERSION
# worker-1   NotReady   <none>          1d    v1.29.0

# Step 2: Gather info
kubectl describe node worker-1 | grep -A5 Conditions
# Look for: KubeletNotReady, NetworkNotReady

# Step 3: SSH to the node
ssh worker-1

# Step 4: Check kubelet
sudo systemctl status kubelet
# Active: inactive (dead)

# Step 5: Check kubelet logs
sudo journalctl -u kubelet -n 50

# Step 6: Fix - Start kubelet
sudo systemctl start kubelet
sudo systemctl enable kubelet

# Step 7: Verify (from master)
kubectl get nodes
# worker-1 should be Ready
```

</details>

---

### Scenario 2: Node Has Disk Pressure

**Objective:** Identify and resolve disk pressure on a node.

**Symptoms:** Node shows `DiskPressure=True` condition.

**Your Task:**
```bash
# Diagnose disk pressure and resolve it
```

<details>
<summary>💡 Solution</summary>

```bash
# Check node conditions
kubectl describe node worker-1 | grep -A10 Conditions
# DiskPressure   True

# SSH to node
ssh worker-1

# Check disk usage
df -h
# Look for filesystems > 85%

# Find large files
sudo du -sh /* 2>/dev/null | sort -hr | head -10

# Common cleanup actions:
# 1. Clean container images
sudo crictl rmi --prune

# 2. Clean old logs
sudo journalctl --vacuum-time=2d

# 3. Remove unused packages
sudo apt autoremove -y

# 4. Clean /var/log
sudo truncate -s 0 /var/log/*.log

# Verify
df -h
kubectl describe node worker-1 | grep DiskPressure
```

</details>

---

### Scenario 3: Node Has Memory Pressure

**Objective:** Handle a node experiencing memory pressure.

**Your Task:**
```bash
# Identify memory usage and mitigate pressure
```

<details>
<summary>💡 Solution</summary>

```bash
# Check node conditions
kubectl describe node worker-1 | grep MemoryPressure

# Check memory usage on node
kubectl top node worker-1

# SSH to node
ssh worker-1
free -h
top -o %MEM

# Find memory-heavy pods
kubectl top pods -A --sort-by=memory

# Options to resolve:
# 1. Evict non-critical pods
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# 2. Cordon node to prevent new pods
kubectl cordon worker-1

# 3. Kill memory-heavy processes on node
# 4. Add memory to VM

# After resolution
kubectl uncordon worker-1
```

</details>

---

### Scenario 4: Node Kernel Issue

**Objective:** Diagnose a node with kernel-related problems.

**Symptoms:** Node shows strange behavior, containers crashing.

**Your Task:**
```bash
# Check kernel and system health
```

<details>
<summary>💡 Solution</summary>

```bash
# SSH to node
ssh worker-1

# Check kernel messages
dmesg | tail -100
sudo dmesg -T | grep -i error

# Check system logs
sudo journalctl -p err -b

# Check kernel parameters for k8s
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward

# If parameters wrong, fix:
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Check if required modules loaded
lsmod | grep br_netfilter
lsmod | grep overlay

# If missing:
sudo modprobe br_netfilter
sudo modprobe overlay
```

</details>

---

### Scenario 5: Kubelet Certificate Expired

**Objective:** Fix kubelet that can't communicate due to expired certs.

**Symptoms:** Node NotReady, kubelet logs show certificate errors.

**Your Task:**
```bash
# Diagnose and fix certificate issues
```

<details>
<summary>💡 Solution</summary>

```bash
# SSH to affected node
ssh worker-1

# Check kubelet logs
sudo journalctl -u kubelet -n 100 | grep -i cert

# Check certificate expiration
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Option 1: Remove node and rejoin
sudo kubeadm reset -f
# On master: generate new join token
kubeadm token create --print-join-command
# Run join command on worker

# Option 2: Rotate certificates (if cluster supports it)
# On master, check certificate rotation config
kubectl get cm -n kube-system kubeadm-config -o yaml | grep rotateCerts

# Force certificate renewal (on master for control plane certs)
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

</details>

---

### Scenario 6: Container Runtime Not Running

**Objective:** Fix a node where containerd is not running.

**Setup:** SSH to worker and stop containerd:
```bash
sudo systemctl stop containerd
```

**Your Task:**
```bash
# Diagnose and fix container runtime issue
```

<details>
<summary>💡 Solution</summary>

```bash
# From master, check node status
kubectl get nodes
kubectl describe node worker-1
# Look for: "container runtime is down"

# SSH to worker
ssh worker-1

# Check containerd status
sudo systemctl status containerd
# Active: inactive (dead)

# Check containerd logs
sudo journalctl -u containerd -n 50

# Start containerd
sudo systemctl start containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo crictl info

# After containerd starts, kubelet should recover
sudo systemctl restart kubelet

# Verify from master
kubectl get nodes
```

</details>

---

### Scenario 7: Worker Node Cannot Join Cluster

**Objective:** Troubleshoot a worker that fails to join the cluster.

**Symptoms:** `kubeadm join` fails with various errors.

**Your Task:**
```bash
# Debug and fix join issues
```

<details>
<summary>💡 Solution</summary>

```bash
# Common Issue 1: Invalid token
# Generate new token on master
kubeadm token create --print-join-command

# Common Issue 2: Port 6443 not reachable
# On worker:
nc -vz <master-ip> 6443
telnet <master-ip> 6443

# Check firewall
sudo iptables -L -n
# Ensure port 6443 is open

# Common Issue 3: Swap enabled
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Common Issue 4: Previous join attempt
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo iptables -F

# Common Issue 5: Wrong CA cert hash
# On master:
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'

# Retry join with correct values
sudo kubeadm join <master>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

</details>

---

### Scenario 8: Drain Node and Handle Pod Eviction

**Objective:** Safely drain a node for maintenance.

**Your Task:**
```bash
# Drain worker-1 for maintenance, handle errors
```

<details>
<summary>💡 Solution</summary>

```bash
# Simple drain (may fail)
kubectl drain worker-1

# Common errors and fixes:

# Error: cannot delete Pods with local storage
kubectl drain worker-1 --delete-emptydir-data

# Error: cannot delete DaemonSet-managed Pods
kubectl drain worker-1 --ignore-daemonsets

# Error: Pod disruption budget (PDB) violation
kubectl drain worker-1 --ignore-daemonsets --force

# Combined command for most cases:
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=30

# Verify node is unschedulable
kubectl get node worker-1
# SchedulingDisabled

# After maintenance
kubectl uncordon worker-1
```

</details>

---

### Scenario 9: Check Node Resource Allocation

**Objective:** Investigate resource allocation and limits on a node.

**Your Task:**
```bash
# Check how resources are allocated on worker-1
```

<details>
<summary>💡 Solution</summary>

```bash
# Check current resource usage
kubectl top node worker-1

# Check allocatable vs capacity
kubectl describe node worker-1 | grep -A7 "Capacity\|Allocatable"

# Check allocated resources (requests/limits)
kubectl describe node worker-1 | grep -A12 "Allocated resources"

# List pods on node with resources
kubectl get pods -A -o wide --field-selector spec.nodeName=worker-1

# Detailed resource view per pod on node
kubectl describe node worker-1 | grep -A50 "Non-terminated Pods"

# Check for resource-related events
kubectl get events -A --field-selector involvedObject.kind=Node
```

</details>

---

### Scenario 10: Node Shows Unknown Status

**Objective:** Handle a node that shows Unknown status.

**Your Task:**
```bash
# Diagnose and recover a node in Unknown state
```

<details>
<summary>💡 Solution</summary>

```bash
# Unknown usually means node-controller lost contact

# Check node status
kubectl get nodes
kubectl describe node worker-1 | grep -A5 Conditions

# Possible causes:
# 1. Network partition between master and worker
# 2. kubelet crashed
# 3. node shut down unexpectedly

# Try SSH to node
ssh worker-1

# If reachable, check kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100

# Restart kubelet
sudo systemctl restart kubelet

# If node is unreachable and needs removal:
kubectl delete node worker-1

# Then either:
# 1. Fix the node and rejoin
# 2. Replace with new node
```

</details>

---

## Control Plane Troubleshooting

### Scenario 11: API Server Not Responding

**Objective:** Diagnose and fix an unresponsive API server.

**Setup:** On master, modify API server manifest to break it:
```bash
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
```

**Your Task:**
```bash
# Fix the API server
```

<details>
<summary>💡 Solution</summary>

```bash
# Symptoms
kubectl get nodes
# The connection to the server was refused

# Since kubectl won't work, check directly on master

# Check if API server container is running
sudo crictl ps | grep kube-apiserver
# No output = not running

# Check static pod manifests
ls -la /etc/kubernetes/manifests/
# kube-apiserver.yaml might be missing or broken

# If missing, restore:
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# If present, check for syntax errors:
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -50

# Check API server logs
sudo crictl logs $(sudo crictl ps -a | grep kube-apiserver | awk '{print $1}')

# Common issues:
# 1. Wrong certificate paths
# 2. etcd not reachable
# 3. Wrong flags/parameters

# After fix, kubelet will restart the pod automatically
# Verify
kubectl get nodes
```

</details>

---

### Scenario 12: Scheduler Not Scheduling Pods

**Objective:** Fix pods stuck in Pending due to scheduler issues.

**Setup:** Break the scheduler:
```bash
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
```

**Your Task:**
```bash
# Fix scheduling issues
```

<details>
<summary>💡 Solution</summary>

```bash
# Create a test pod
kubectl run test-pod --image=nginx

# Check status
kubectl get pods
# NAME       READY   STATUS    RESTARTS   AGE
# test-pod   0/1     Pending   0          1m

# Check events
kubectl describe pod test-pod
# Events: <none> or no scheduler events

# Check scheduler
kubectl get pods -n kube-system | grep scheduler
# No scheduler pod running

# Check manifest
ls /etc/kubernetes/manifests/
# kube-scheduler.yaml missing!

# Restore scheduler manifest
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

# Wait for scheduler to start
kubectl get pods -n kube-system | grep scheduler

# Verify pod gets scheduled
kubectl get pod test-pod
# STATUS: Running
```

</details>

---

### Scenario 13: Controller Manager Not Working

**Objective:** Fix replica issues caused by broken controller manager.

**Setup:** Create a deployment, then break controller manager
```bash
kubectl create deployment nginx-deploy --image=nginx --replicas=3
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
```

**Your Task:**
```bash
# ReplicaSet not maintaining desired replicas - diagnose and fix
```

<details>
<summary>💡 Solution</summary>

```bash
# Delete a pod from deployment
kubectl delete pod $(kubectl get pods -l app=nginx-deploy -o name | head -1)

# Check if pod is recreated
kubectl get pods -l app=nginx-deploy
# Only 2 pods! No new pod created

# Check controller manager
kubectl get pods -n kube-system | grep controller-manager
# No controller-manager pod!

# Restore manifest
sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/

# Wait for it to start
kubectl get pods -n kube-system | grep controller-manager

# Verify replicas are maintained
kubectl get pods -l app=nginx-deploy
# 3 pods running

# Check controller manager logs for any issues
kubectl logs -n kube-system kube-controller-manager-<master-name>
```

</details>

---

### Scenario 14: etcd Cluster Unhealthy

**Objective:** Diagnose and fix etcd issues.

**Your Task:**
```bash
# Check etcd health and fix issues
```

<details>
<summary>💡 Solution</summary>

```bash
# Check etcd pod status
kubectl get pods -n kube-system | grep etcd

# Check etcd member health
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check member list
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Check etcd logs
kubectl logs -n kube-system etcd-<master-name>

# Common fixes:
# 1. Restart etcd pod
kubectl delete pod -n kube-system etcd-<master-name>

# 2. Check disk space on etcd data dir
df -h /var/lib/etcd

# 3. Defragment etcd (if database too large)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag
```

</details>

---

### Scenario 15: etcd Backup and Restore

**Objective:** Backup etcd and restore from backup.

**Your Task:**
```bash
# Take etcd backup and restore from it
```

<details>
<summary>💡 Solution</summary>

```bash
# Step 1: Backup etcd
sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table

# Step 2: Restore (simulating disaster recovery)
# Stop kube-apiserver first
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore to new data directory
sudo ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Update etcd manifest to use new data dir
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' \
  /etc/kubernetes/manifests/etcd.yaml

# Restart control plane
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Verify cluster works
kubectl get nodes
kubectl get pods -A
```

**Exam Tip:** Remember the cert paths! They're in `/etc/kubernetes/pki/etcd/`

</details>

---

### Scenario 16: Kube-Proxy Not Working

**Objective:** Fix kube-proxy issues affecting service connectivity.

**Your Task:**
```bash
# Services not reachable - diagnose kube-proxy
```

<details>
<summary>💡 Solution</summary>

```bash
# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Check logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50

# Check kube-proxy mode
kubectl get cm -n kube-system kube-proxy -o yaml | grep mode

# Check iptables rules (if iptables mode)
sudo iptables -t nat -L KUBE-SERVICES | head -20

# Check IPVS (if ipvs mode)
sudo ipvsadm -L -n

# Common fixes:
# 1. Restart kube-proxy
kubectl rollout restart daemonset kube-proxy -n kube-system

# 2. Check ConfigMap
kubectl edit cm kube-proxy -n kube-system

# 3. Verify clusterCIDR matches pod network
kubectl get cm kube-proxy -n kube-system -o yaml | grep clusterCIDR
```

</details>

---

### Scenario 17: Control Plane Certificate Issues

**Objective:** Check and renew control plane certificates.

**Your Task:**
```bash
# Check certificate expiration and renew if needed
```

<details>
<summary>💡 Solution</summary>

```bash
# Check all certificate expirations
sudo kubeadm certs check-expiration

# Output shows:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Jan 01, 2026 00:00 UTC   364d
# apiserver                  Jan 01, 2026 00:00 UTC   364d
# ...

# Renew all certificates
sudo kubeadm certs renew all

# After renewal, restart control plane components
sudo systemctl restart kubelet

# Update kubeconfig files
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
```

</details>

---

### Scenario 18: Static Pod Not Starting

**Objective:** Debug a static pod that won't start.

**Setup:** Create a broken static pod manifest
```bash
cat <<EOF | sudo tee /etc/kubernetes/manifests/broken-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-test
spec:
  containers:
  - name: nginx
    image: nginx:nonexistent-tag
EOF
```

**Your Task:**
```bash
# Fix the static pod
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pods -A | grep static

# Describe pod
kubectl describe pod static-test-<node-name>
# Events show: ImagePullBackOff

# Static pods are managed by kubelet directly
# Edit the manifest file

sudo vi /etc/kubernetes/manifests/broken-pod.yaml
# Change: image: nginx:nonexistent-tag
# To: image: nginx:latest

# Kubelet automatically recreates the pod
# Verify
kubectl get pods | grep static
# STATUS: Running

# To delete static pod, remove manifest file
sudo rm /etc/kubernetes/manifests/broken-pod.yaml
```

</details>

---

## Application Troubleshooting

### Scenario 19: Pod CrashLoopBackOff

**Objective:** Debug a pod that keeps crashing.

**Setup:**
```bash
kubectl run crasher --image=busybox --restart=Always -- /bin/sh -c "exit 1"
```

**Your Task:**
```bash
# Fix the crashing pod
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod crasher
# STATUS: CrashLoopBackOff

# Check logs
kubectl logs crasher
# No output (exits immediately)

kubectl logs crasher --previous
# Check previous container logs

# Describe pod
kubectl describe pod crasher
# Restart Count: 5
# Last State: Terminated - Exit Code: 1

# Check command
kubectl get pod crasher -o yaml | grep -A5 "command\|args"
# Command exits with code 1

# Fix: Update command to keep running
kubectl delete pod crasher
kubectl run crasher --image=busybox --restart=Always -- /bin/sh -c "sleep 3600"

# Verify
kubectl get pod crasher
# STATUS: Running
```

</details>

---

### Scenario 20: Pod ImagePullBackOff

**Objective:** Fix an image pull failure.

**Setup:**
```bash
kubectl run badimage --image=nginx:doesnotexist
```

**Your Task:**
```bash
# Fix the image pull issue
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod badimage
# STATUS: ImagePullBackOff

# Describe pod for details
kubectl describe pod badimage | grep -A10 Events
# Failed to pull image "nginx:doesnotexist"
# Error: manifest unknown

# Fix: Update image to valid tag
kubectl set image pod/badimage badimage=nginx:latest

# Or delete and recreate
kubectl delete pod badimage
kubectl run badimage --image=nginx:latest

# Verify
kubectl get pod badimage
# STATUS: Running
```

**Common Image Issues:**
- Wrong image name or tag
- Private registry without imagePullSecrets
- Network connectivity to registry
- Rate limiting (Docker Hub)

</details>

---

### Scenario 21: Pod Stuck in Pending - No Resources

**Objective:** Fix a pod pending due to insufficient resources.

**Setup:**
```bash
kubectl run bigpod --image=nginx --requests=cpu=100,memory=100Gi
```

**Your Task:**
```bash
# Diagnose why pod is pending
```

<details>
<summary>💡 Solution</summary>

```bash
# Check status
kubectl get pod bigpod
# STATUS: Pending

# Describe pod
kubectl describe pod bigpod
# Events: 0/3 nodes are available: 3 Insufficient memory

# Check requested vs available
kubectl get pod bigpod -o yaml | grep -A5 resources
kubectl describe nodes | grep -A5 "Allocated resources"

# Fix: Reduce resource request
kubectl delete pod bigpod
kubectl run bigpod --image=nginx --requests=cpu=100m,memory=128Mi

# Or add node with more resources
# Verify
kubectl get pod bigpod
```

</details>

---

### Scenario 22: Pod Stuck in Pending - Node Selector

**Objective:** Fix a pod with unsatisfiable node selector.

**Setup:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    disktype: ssd
EOF
```

**Your Task:**
```bash
# Fix the pod scheduling issue
```

<details>
<summary>💡 Solution</summary>

```bash
# Check status
kubectl get pod ssd-pod
# STATUS: Pending

# Describe
kubectl describe pod ssd-pod
# node(s) didn't match Pod's node affinity/selector

# Check node labels
kubectl get nodes --show-labels | grep disktype
# No nodes have disktype=ssd

# Option 1: Add label to a node
kubectl label node worker-1 disktype=ssd

# Option 2: Remove nodeSelector
kubectl delete pod ssd-pod
kubectl run ssd-pod --image=nginx

# Verify
kubectl get pod ssd-pod -o wide
```

</details>

---

### Scenario 23: Pod Stuck Due to Taints

**Objective:** Fix a pod that can't schedule due to taints.

**Setup:**
```bash
kubectl taint nodes worker-1 dedicated=special:NoSchedule
kubectl taint nodes worker-2 dedicated=special:NoSchedule
kubectl run taint-test --image=nginx
```

**Your Task:**
```bash
# Pod is pending - fix the taint issue
```

<details>
<summary>💡 Solution</summary>

```bash
# Check status
kubectl describe pod taint-test | grep -A5 Events
# 0/2 nodes are available: 2 node(s) had untolerated taint

# Check taints on nodes
kubectl describe nodes | grep Taints

# Option 1: Add toleration to pod
kubectl delete pod taint-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: taint-test
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "special"
    effect: "NoSchedule"
EOF

# Option 2: Remove taint from node
kubectl taint nodes worker-1 dedicated=special:NoSchedule-
kubectl taint nodes worker-2 dedicated=special:NoSchedule-

# Verify
kubectl get pod taint-test -o wide
```

</details>

---

### Scenario 24: Application OOMKilled

**Objective:** Debug an application killed due to memory limits.

**Setup:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
    resources:
      limits:
        memory: "100Mi"
EOF
```

**Your Task:**
```bash
# Diagnose and fix OOMKilled issue
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod oom-pod
# STATUS: OOMKilled or CrashLoopBackOff

# Describe pod
kubectl describe pod oom-pod
# Last State: Terminated - Reason: OOMKilled

# Check resource limits
kubectl get pod oom-pod -o yaml | grep -A5 limits
# memory: 100Mi (but app needs 250M)

# Fix: Increase memory limit
kubectl delete pod oom-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
    resources:
      limits:
        memory: "300Mi"
      requests:
        memory: "256Mi"
EOF

# Verify
kubectl get pod oom-pod
```

</details>

---

### Scenario 25: Liveness Probe Failing

**Objective:** Fix a pod restarting due to liveness probe failures.

**Setup:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
spec:
  containers:
  - name: app
    image: nginx
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
EOF
```

**Your Task:**
```bash
# Fix the liveness probe issue
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod
kubectl get pod liveness-test
# Restarts keep increasing

# Describe
kubectl describe pod liveness-test
# Liveness probe failed: connection refused

# Problem: nginx serves on port 80, not 8080
# And /healthz doesn't exist in default nginx

# Fix: Correct the probe
kubectl delete pod liveness-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
spec:
  containers:
  - name: app
    image: nginx
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

# Verify no restarts
kubectl get pod liveness-test -w
```

</details>

---

### Scenario 26: Readiness Probe Failing

**Objective:** Debug a pod not receiving traffic due to readiness probe.

**Setup:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ready-test
  labels:
    app: ready-test
spec:
  containers:
  - name: app
    image: nginx
    readinessProbe:
      exec:
        command: ["cat", "/tmp/ready"]
      initialDelaySeconds: 5
      periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ready-svc
spec:
  selector:
    app: ready-test
  ports:
  - port: 80
EOF
```

**Your Task:**
```bash
# Pod not receiving traffic - fix readiness
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod ready-test
# READY: 0/1 (not ready!)

# Check endpoints
kubectl get endpoints ready-svc
# ENDPOINTS: <none>

# Describe pod
kubectl describe pod ready-test
# Readiness probe failed: cat: /tmp/ready: No such file

# Fix: Create the file the probe expects
kubectl exec ready-test -- touch /tmp/ready

# Verify pod becomes ready
kubectl get pod ready-test
# READY: 1/1

# Verify endpoints
kubectl get endpoints ready-svc
# ENDPOINTS: 10.244.x.x:80
```

</details>

---

### Scenario 27: Multi-Container Pod Issues

**Objective:** Debug a multi-container pod with failing sidecar.

**Setup:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: main
    image: nginx
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'exit 1']
EOF
```

**Your Task:**
```bash
# Fix the failing sidecar container
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod multi-container
# READY: 1/2

# Check specific container status
kubectl describe pod multi-container | grep -A10 "sidecar:"
# State: Waiting - CrashLoopBackOff

# Check sidecar logs
kubectl logs multi-container -c sidecar
# (empty - exits immediately)

# Fix: Update sidecar command
kubectl delete pod multi-container
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: main
    image: nginx
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo sidecar running; sleep 60; done']
EOF

# Verify
kubectl get pod multi-container
# READY: 2/2
```

</details>

---

### Scenario 28: Init Container Stuck

**Objective:** Fix a pod stuck in Init state.

**Setup:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: init-wait
    image: busybox
    command: ['sh', '-c', 'until nslookup nonexistent-service; do sleep 2; done']
  containers:
  - name: main
    image: nginx
EOF
```

**Your Task:**
```bash
# Fix the stuck init container
```

<details>
<summary>💡 Solution</summary>

```bash
# Check pod status
kubectl get pod init-pod
# STATUS: Init:0/1

# Check init container logs
kubectl logs init-pod -c init-wait
# nslookup: can't resolve 'nonexistent-service'

# The init container is waiting for a service that doesn't exist

# Option 1: Create the expected service
kubectl create service clusterip nonexistent-service --tcp=80:80

# Option 2: Fix the init container command
kubectl delete pod init-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: init-wait
    image: busybox
    command: ['sh', '-c', 'echo "Init complete"']
  containers:
  - name: main
    image: nginx
EOF

# Verify
kubectl get pod init-pod
# STATUS: Running
```

</details>

---

## Networking Troubleshooting

### Scenario 29: Service Not Reachable

**Objective:** Debug a service that pods cannot reach.

**Setup:**
```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
kubectl run test-client --image=busybox --restart=Never -- sleep 3600
```

**Your Task:**
```bash
# test-client cannot reach web service - diagnose
```

<details>
<summary>💡 Solution</summary>

```bash
# Test connectivity
kubectl exec test-client -- wget -O- -T5 web
# If fails, start debugging

# Step 1: Check service exists
kubectl get svc web
kubectl describe svc web

# Step 2: Check endpoints
kubectl get endpoints web
# If empty, labels don't match

# Step 3: Check pod labels vs service selector
kubectl get pods -l app=web
kubectl get svc web -o jsonpath='{.spec.selector}'

# Step 4: Check if pods are ready
kubectl get pods -l app=web -o wide

# Step 5: Test direct pod IP
kubectl exec test-client -- wget -O- -T5 <pod-ip>

# Common fixes:
# 1. Fix selector mismatch
kubectl patch svc web -p '{"spec":{"selector":{"app":"web"}}}'

# 2. Fix target port
kubectl patch svc web -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'

# Verify
kubectl exec test-client -- wget -O- -T5 web
```

</details>

---

### Scenario 30: DNS Not Resolving

**Objective:** Fix DNS resolution issues in the cluster.

**Your Task:**
```bash
# Pods cannot resolve service names - diagnose DNS
```

<details>
<summary>💡 Solution</summary>

```bash
# Test DNS resolution
kubectl exec test-client -- nslookup kubernetes.default
# If fails: "nslookup: can't resolve"

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
# All pods should be Running

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Check CoreDNS service
kubectl get svc -n kube-system kube-dns
# ClusterIP should be 10.96.0.10 (default)

# Check CoreDNS endpoints
kubectl get endpoints -n kube-system kube-dns

# Check CoreDNS ConfigMap
kubectl get cm -n kube-system coredns -o yaml

# Common fixes:
# 1. Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# 2. Fix ConfigMap if corrupted
kubectl edit cm coredns -n kube-system

# 3. Check if pod can reach CoreDNS
kubectl exec test-client -- nc -vz 10.96.0.10 53

# Verify
kubectl exec test-client -- nslookup web.default.svc.cluster.local
```

</details>

---

### Scenario 31: Network Policy Blocking Traffic

**Objective:** Debug traffic blocked by network policy.

**Setup:**
```bash
# Create pods and blocking policy
kubectl run web --image=nginx --labels=app=web --expose --port=80
kubectl run client --image=busybox --labels=app=client -- sleep 3600
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
EOF
```

**Your Task:**
```bash
# client cannot reach web - diagnose
```

<details>
<summary>💡 Solution</summary>

```bash
# Test connectivity
kubectl exec client -- wget -O- -T5 web
# Timeout - blocked

# Check network policies
kubectl get networkpolicy
kubectl describe networkpolicy deny-all
# Pod Selector: app=web
# Allowing ingress: <none> (default deny)

# Fix: Allow traffic from client
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - port: 80
EOF

# Verify
kubectl exec client -- wget -O- -T5 web
# Should work now!
```

</details>

---

### Scenario 32: NodePort Service Not Accessible

**Objective:** Fix a NodePort service not reachable from outside.

**Setup:**
```bash
kubectl create deployment nodeport-app --image=nginx
kubectl expose deployment nodeport-app --type=NodePort --port=80
```

**Your Task:**
```bash
# Cannot reach service from outside cluster
```

<details>
<summary>💡 Solution</summary>

```bash
# Get NodePort
kubectl get svc nodeport-app
# PORT: 80:31234/TCP (31234 is NodePort)

# Check from within cluster
kubectl run tmp --rm -it --image=busybox -- wget -O- nodeport-app

# Check from node (SSH to any node)
curl localhost:31234

# If external access fails, check:

# 1. Security Group / Firewall
# Allow ports 30000-32767 from your IP

# 2. Node External IP
kubectl get nodes -o wide
# Use EXTERNAL-IP with NodePort

# 3. kube-proxy running
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# 4. iptables rules exist
sudo iptables -t nat -L | grep 31234

# Test from your machine
curl http://<node-external-ip>:31234
```

</details>

---

### Scenario 33: Ingress Not Routing

**Objective:** Fix ingress not routing traffic to services.

**Setup:**
```bash
kubectl create deployment app1 --image=nginx
kubectl expose deployment app1 --port=80
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
EOF
```

**Your Task:**
```bash
# Ingress not working - diagnose
```

<details>
<summary>💡 Solution</summary>

```bash
# Check Ingress
kubectl get ingress test-ingress
kubectl describe ingress test-ingress

# Check if Ingress Controller is installed
kubectl get pods -n ingress-nginx
# If no controller, install one:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Wait for controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Check Ingress has ADDRESS
kubectl get ingress test-ingress
# ADDRESS should be populated

# Test with curl
curl -H "Host: app.example.com" http://<ingress-ip>

# Common issues:
# 1. Missing ingressClassName
kubectl patch ingress test-ingress -p '{"spec":{"ingressClassName":"nginx"}}'

# 2. Service name/port wrong
# 3. Backend service not running
```

</details>

---

### Scenario 34: Pod Cannot Reach External Network

**Objective:** Fix pods unable to reach the internet.

**Your Task:**
```bash
# Pods cannot reach external URLs
```

<details>
<summary>💡 Solution</summary>

```bash
# Test external connectivity
kubectl exec test-client -- wget -O- -T5 http://google.com
# Fails

# Check DNS for external names
kubectl exec test-client -- nslookup google.com

# If DNS works but connection fails:

# 1. Check node can reach external
ssh worker-1
curl google.com

# 2. Check NAT rules
sudo iptables -t nat -L POSTROUTING

# 3. Check CNI configuration
ls /etc/cni/net.d/

# 4. Check if egress Network Policy blocks
kubectl get networkpolicy -A

# 5. Verify ip_forward is enabled
cat /proc/sys/net/ipv4/ip_forward
# Should be 1

# If 0, enable it:
sudo sysctl -w net.ipv4.ip_forward=1
```

</details>

---

### Scenario 35: Pod-to-Pod Communication Failing

**Objective:** Fix pods that cannot communicate with each other.

**Your Task:**
```bash
# Pods on different nodes cannot communicate
```

<details>
<summary>💡 Solution</summary>

```bash
# Create pods on different nodes
kubectl run pod1 --image=nginx --overrides='{"spec":{"nodeName":"worker-1"}}'
kubectl run pod2 --image=busybox --overrides='{"spec":{"nodeName":"worker-2"}}' -- sleep 3600

# Get pod1 IP
kubectl get pod pod1 -o wide

# Test from pod2
kubectl exec pod2 -- wget -O- -T5 <pod1-ip>

# If fails, check CNI:

# 1. CNI pods running
kubectl get pods -n kube-flannel  # or kube-system for other CNIs

# 2. CNI config on nodes
ssh worker-1
ls /etc/cni/net.d/

# 3. Check routes
ip route

# 4. Reinstall CNI if needed
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# 5. Check iptables
sudo iptables -L FORWARD
```

</details>

---

### Scenario 36: Service Endpoints Empty

**Objective:** Fix a service with no endpoints.

**Setup:**
```bash
kubectl run mypod --image=nginx --labels=run=mypod
kubectl create service clusterip mysvc --tcp=80:80
```

**Your Task:**
```bash
# Service has no endpoints
```

<details>
<summary>💡 Solution</summary>

```bash
# Check endpoints
kubectl get endpoints mysvc
# ENDPOINTS: <none>

# Check service selector
kubectl get svc mysvc -o jsonpath='{.spec.selector}'
# Output: {"app":"mysvc"}

# Check pod labels
kubectl get pod mypod --show-labels
# Labels: run=mypod

# Mismatch! Service selects app=mysvc, pod has run=mypod

# Fix Option 1: Update pod label
kubectl label pod mypod app=mysvc

# Fix Option 2: Update service selector
kubectl patch svc mysvc -p '{"spec":{"selector":{"run":"mypod"}}}'

# Verify
kubectl get endpoints mysvc
# ENDPOINTS: 10.244.x.x:80
```

</details>

---

### Scenario 37: Debug with Ephemeral Containers

**Objective:** Use ephemeral debug container to troubleshoot.

**Your Task:**
```bash
# Debug a distroless container with no shell
```

<details>
<summary>💡 Solution</summary>

```bash
# Create a distroless pod (no shell)
kubectl run distroless --image=gcr.io/distroless/static-debian11 -- /pause

# Try to exec - fails
kubectl exec distroless -- sh
# error: Internal error occurred: unable to upgrade connection

# Use ephemeral container
kubectl debug distroless -it --image=busybox --target=distroless

# Now you have a shell in the same pod namespace
# Can inspect:
# - Network namespace (same IP)
# - Process namespace
# - File system (if using --share-processes)

# Alternative: Create debug pod copying target pod's namespace
kubectl debug distroless -it --image=busybox --copy-to=debug-pod

# Debug node directly
kubectl debug node/worker-1 -it --image=busybox
# chroot /host to access node filesystem
```

</details>

---

### Scenario 38: Full Cluster Connectivity Test

**Objective:** Comprehensive network troubleshooting exercise.

**Your Task:**
```bash
# Perform complete network diagnostics
```

<details>
<summary>💡 Solution</summary>

```bash
#!/bin/bash
# Complete connectivity test script

echo "=== Cluster Network Diagnostics ==="

# 1. Check nodes
echo -e "\n[1] Node Status:"
kubectl get nodes -o wide

# 2. Check CNI
echo -e "\n[2] CNI Status:"
kubectl get pods -A | grep -E "flannel|calico|weave|cilium"

# 3. Check CoreDNS
echo -e "\n[3] CoreDNS Status:"
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 4. Check kube-proxy
echo -e "\n[4] kube-proxy Status:"
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# 5. Create test pods
echo -e "\n[5] Creating test pods..."
kubectl run net-test-1 --image=busybox --restart=Never -- sleep 3600
kubectl run net-test-2 --image=nginx --restart=Never
kubectl expose pod net-test-2 --port=80 --name=test-svc
sleep 10

# 6. Test DNS
echo -e "\n[6] DNS Test:"
kubectl exec net-test-1 -- nslookup kubernetes.default

# 7. Test service
echo -e "\n[7] Service Test:"
kubectl exec net-test-1 -- wget -O- -T5 test-svc

# 8. Test external
echo -e "\n[8] External Network Test:"
kubectl exec net-test-1 -- wget -O- -T5 google.com 2>/dev/null | head -5

# 9. Cleanup
echo -e "\n[9] Cleanup:"
kubectl delete pod net-test-1 net-test-2 --force --grace-period=0 2>/dev/null
kubectl delete svc test-svc 2>/dev/null

echo -e "\n=== Diagnostics Complete ==="
```

</details>

---

## Quick Troubleshooting Reference

### Pod States Cheat Sheet

| State | Meaning | Debug Commands |
|-------|---------|----------------|
| Pending | Not scheduled | `describe pod`, check resources/taints |
| ContainerCreating | Image pulling or volume mounting | `describe pod`, check events |
| Running | Container started | Check if working: `logs`, `exec` |
| CrashLoopBackOff | Keeps crashing | `logs --previous`, fix command/image |
| ImagePullBackOff | Can't pull image | Check image name, registry access |
| OOMKilled | Out of memory | Increase memory limits |
| Terminating | Being deleted | Check finalizers, force delete |

### Control Plane Debug Commands

```bash
# Check component status
kubectl get componentstatuses  # deprecated but useful

# Static pod manifests
ls /etc/kubernetes/manifests/

# Check component logs
kubectl logs -n kube-system <pod-name>

# Kubelet logs
sudo journalctl -u kubelet -n 100

# etcd health
sudo ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Exam Tips for Troubleshooting

1. **Start broad, then narrow:** `kubectl get pods -A` → specific pod → specific container
2. **Always check events:** `kubectl describe` shows events at the bottom
3. **Use previous logs:** `kubectl logs --previous` for crashed containers
4. **Know static pod locations:** `/etc/kubernetes/manifests/`
5. **Know log locations:** `journalctl -u kubelet`, `/var/log/pods/`
6. **Practice under time pressure:** Set 5-minute limits per scenario

---

**Next:** Move to [03-WORKLOADS-SCHEDULING-SCENARIOS.md](./03-WORKLOADS-SCHEDULING-SCENARIOS.md)
