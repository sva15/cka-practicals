# CKA Exam Intensive 8-Day Roadmap 🎯

> **Exam Date Target:** January 2, 2025 (or shortly after)  
> **Study Period:** December 25, 2024 - January 1, 2025  
> **Daily Commitment:** 14 hours/day  
> **Total Study Hours:** 112 hours  
> **Approach:** 40% Theory + 60% Hands-on

---

## Table of Contents
1. [Lab Environment Setup (AWS EC2 + kubeadm)](#lab-environment-setup)
2. [Daily Schedule Template](#daily-schedule-template)
3. [8-Day Study Plan Overview](#8-day-study-plan-overview)
4. [Day-by-Day Detailed Breakdown](#day-by-day-detailed-breakdown)
5. [Resources & Documentation](#resources--documentation)

---

## Lab Environment Setup

### AWS EC2 Infrastructure Requirements

You'll need **3 EC2 instances** for a production-like kubeadm cluster:

| Node | Role | Instance Type | Storage | Purpose |
|------|------|---------------|---------|---------|
| master-1 | Control Plane | t3.medium (2 vCPU, 4GB RAM) | 30GB gp3 | API Server, etcd, Scheduler, Controller Manager |
| worker-1 | Worker Node | t3.medium | 30GB gp3 | Run workloads |
| worker-2 | Worker Node | t3.medium | 30GB gp3 | Run workloads, test HA scenarios |

> **Cost Estimate:** ~$3-5/day for all 3 instances (stop when not studying!)

### Step 1: Launch EC2 Instances

```bash
# Launch 3 Ubuntu 22.04 LTS instances with these settings:
# - AMI: Ubuntu Server 22.04 LTS (HVM)
# - Instance Type: t3.medium
# - Key Pair: Create or select existing
# - Security Group: Allow ports 22, 6443, 2379-2380, 10250-10259, 30000-32767
# - Storage: 30GB gp3
```

**Security Group Rules:**

| Type | Port Range | Source | Purpose |
|------|------------|--------|---------|
| SSH | 22 | Your IP | SSH access |
| Custom TCP | 6443 | 0.0.0.0/0 | Kubernetes API |
| Custom TCP | 2379-2380 | VPC CIDR | etcd |
| Custom TCP | 10250-10259 | VPC CIDR | Kubelet, Scheduler, Controller |
| Custom TCP | 30000-32767 | 0.0.0.0/0 | NodePort Services |

### Step 2: Prepare All Nodes (Run on ALL 3 nodes)

```bash
#!/bin/bash
# Run this script on ALL nodes (master and workers)

# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Disable swap (Required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install containerd
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "Node preparation complete!"
```

### Step 3: Initialize Control Plane (Master Node ONLY)

```bash
# On master-1 ONLY
# Replace <MASTER_PRIVATE_IP> with your master's private IP

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_PRIVATE_IP>

# Setup kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Save the join command output! You'll need it for worker nodes
```

### Step 4: Install CNI (Flannel)

```bash
# On master-1
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify CNI is running
kubectl get pods -n kube-flannel
```

### Step 5: Join Worker Nodes

```bash
# On worker-1 and worker-2
# Use the join command from kubeadm init output
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>
```

### Step 6: Verify Cluster

```bash
# On master-1
kubectl get nodes
# All nodes should be "Ready" status

kubectl get pods -A
# All system pods should be "Running"
```

### Quick Cluster Reset (When Needed)

```bash
# If you need to reset and start fresh:
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo iptables -F && sudo iptables -t nat -F
```

---

## Daily Schedule Template

### 14-Hour Study Day Structure

| Time Block | Duration | Activity | Type |
|------------|----------|----------|------|
| 06:00 - 06:30 | 30 min | Review previous day notes | Theory |
| 06:30 - 09:00 | 2.5 hrs | **Morning Theory Session** | Theory |
| 09:00 - 09:30 | 30 min | Break + Breakfast | Rest |
| 09:30 - 12:30 | 3 hrs | **Hands-on Lab Session 1** | Practical |
| 12:30 - 13:30 | 1 hr | Lunch Break | Rest |
| 13:30 - 16:00 | 2.5 hrs | **Hands-on Lab Session 2** | Practical |
| 16:00 - 16:30 | 30 min | Break | Rest |
| 16:30 - 19:00 | 2.5 hrs | **Hands-on Lab Session 3** | Practical |
| 19:00 - 20:00 | 1 hr | Dinner Break | Rest |
| 20:00 - 21:30 | 1.5 hrs | **Evening Theory + Documentation** | Theory |
| 21:30 - 22:00 | 30 min | Daily Review + Tomorrow Planning | Review |

**Total:** 14 hours study + 3 hours breaks = 17 hours active day

---

## 8-Day Study Plan Overview

| Day | Date | Focus Domain(s) | Weight | Hours |
|-----|------|-----------------|--------|-------|
| **Day 1** | Dec 25 | Cluster Architecture: RBAC, kubeadm basics | 25% | 14 hrs |
| **Day 2** | Dec 26 | Cluster Architecture: etcd, HA, Upgrades + Storage basics | 25%+10% | 14 hrs |
| **Day 3** | Dec 27 | Cluster Architecture: Helm, Kustomize, CRDs + Storage deep-dive | 25%+10% | 14 hrs |
| **Day 4** | Dec 28 | Services & Networking: Services, DNS, Network Policies | 20% | 14 hrs |
| **Day 5** | Dec 29 | Services & Networking: Ingress, Gateway API + Workloads basics | 20%+15% | 14 hrs |
| **Day 6** | Dec 30 | Workloads & Scheduling + Troubleshooting basics | 15%+30% | 14 hrs |
| **Day 7** | Dec 31 | Troubleshooting intensive | 30% | 14 hrs |
| **Day 8** | Jan 1 | Mock Exams + Weak Area Review | All | 14 hrs |

---

## Day-by-Day Detailed Breakdown

---

## 📅 DAY 1 - December 25 (Wednesday)
### Focus: Cluster Architecture - RBAC & kubeadm Foundations

#### 🎯 Learning Objectives
- [ ] Master RBAC concepts and implementation
- [ ] Understand kubeadm cluster initialization
- [ ] Practice creating Roles, ClusterRoles, and Bindings

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **Kubernetes Architecture Overview** (1 hour)
   - Control plane components (API Server, etcd, Scheduler, Controller Manager)
   - Node components (kubelet, kube-proxy, container runtime)
   - Read: [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

2. **RBAC Deep Dive** (1.5 hours)
   - Subjects: Users, Groups, ServiceAccounts
   - Resources: Roles, ClusterRoles, RoleBindings, ClusterRoleBindings
   - Verbs: get, list, watch, create, update, patch, delete
   - Read: [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

**Evening (20:00 - 21:30)**
3. **kubeadm Architecture** (1.5 hours)
   - Certificate management
   - Static pod manifests
   - Bootstrap tokens
   - Read: [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): RBAC Basics**
- [ ] Create a ServiceAccount for a developer
- [ ] Create a Role with limited permissions
- [ ] Create a RoleBinding
- [ ] Test permissions using `kubectl auth can-i`
- [ ] Create a ClusterRole for read-only access
- [ ] Practice with namespace-scoped vs cluster-scoped resources

**Lab Session 2 (13:30 - 16:00): Advanced RBAC**
- [ ] Create a ClusterRole for viewing all resources
- [ ] Aggregate ClusterRoles
- [ ] Debug RBAC issues using `kubectl auth can-i --list`
- [ ] Create service account tokens
- [ ] Configure kubeconfig for service accounts

**Lab Session 3 (16:30 - 19:00): kubeadm Operations**
- [ ] Examine control plane static pod manifests
- [ ] Explore certificate locations and contents
- [ ] Practice `kubeadm token create`
- [ ] Simulate adding a new worker node
- [ ] Explore kubeadm phases

#### ✅ Day 1 Checkpoints
- [ ] Can create and test RBAC policies
- [ ] Understand difference between Role and ClusterRole
- [ ] Know kubeadm configuration locations
- [ ] Can generate join tokens

---

## 📅 DAY 2 - December 26 (Thursday)
### Focus: etcd, High Availability, Cluster Upgrades + Storage Basics

#### 🎯 Learning Objectives
- [ ] Master etcd backup and restore procedures
- [ ] Understand HA control plane configurations
- [ ] Perform cluster version upgrades
- [ ] Understand storage fundamentals

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **etcd Fundamentals** (1.5 hours)
   - etcd architecture and data model
   - etcdctl commands
   - Backup and restore strategies
   - Read: [Operating etcd clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

2. **High Availability Concepts** (1 hour)
   - Stacked vs External etcd topology
   - Load balancer requirements
   - Leader election
   - Read: [HA Considerations](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

**Evening (20:00 - 21:30)**
3. **Cluster Upgrades** (1 hour)
   - Upgrade strategy: control plane first, then workers
   - kubeadm upgrade workflow
   - Read: [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

4. **Storage Fundamentals** (1 hour)
   - Volumes, PersistentVolumes, PersistentVolumeClaims
   - StorageClasses
   - Access modes and reclaim policies

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): etcd Operations**
- [ ] Install etcdctl
- [ ] Take etcd snapshot: `etcdctl snapshot save`
- [ ] Verify snapshot: `etcdctl snapshot status`
- [ ] Restore from snapshot to a new location
- [ ] Practice with etcd certificates authentication
- [ ] Explore etcd data with `etcdctl get`

**Lab Session 2 (13:30 - 16:00): Cluster Upgrades**
- [ ] Check current cluster version
- [ ] Plan upgrade path
- [ ] Drain control plane node
- [ ] Upgrade kubeadm, kubelet, kubectl on control plane
- [ ] Uncordon and verify
- [ ] Repeat for worker nodes

**Lab Session 3 (16:30 - 19:00): Storage Basics**
- [ ] Create hostPath volumes
- [ ] Create emptyDir volumes
- [ ] Create PersistentVolume (local)
- [ ] Create PersistentVolumeClaim
- [ ] Mount PVC in a Pod
- [ ] Test data persistence

#### ✅ Day 2 Checkpoints
- [ ] Can backup and restore etcd
- [ ] Understand HA topology options
- [ ] Can perform cluster upgrades
- [ ] Understand PV/PVC lifecycle

---

## 📅 DAY 3 - December 27 (Friday)
### Focus: Helm, Kustomize, CRDs, Extensions + Storage Deep-dive

#### 🎯 Learning Objectives
- [ ] Use Helm to install and manage applications
- [ ] Use Kustomize for configuration management
- [ ] Understand CRDs and Operators
- [ ] Master dynamic volume provisioning

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **Helm Deep Dive** (1.5 hours)
   - Helm architecture: charts, releases, repositories
   - helm install, upgrade, rollback
   - Values and templating basics
   - Read: [Helm Documentation](https://helm.sh/docs/)

2. **Kustomize** (1 hour)
   - Bases and overlays
   - Patches and generators
   - Read: [Kustomize Documentation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)

**Evening (20:00 - 21:30)**
3. **Extension Interfaces** (1 hour)
   - CNI (Container Network Interface)
   - CSI (Container Storage Interface)
   - CRI (Container Runtime Interface)
   - Read: [Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/)

4. **CRDs and Operators** (1 hour)
   - Custom Resource Definitions
   - Operator pattern
   - Installing operators

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): Helm**
- [ ] Install Helm
- [ ] Add helm repositories
- [ ] Search and install charts
- [ ] Customize installations with values
- [ ] Upgrade releases
- [ ] Rollback releases
- [ ] Uninstall releases

**Lab Session 2 (13:30 - 16:00): Kustomize + CRDs**
- [ ] Create base configurations
- [ ] Create environment overlays
- [ ] Use patches for modifications
- [ ] Apply with `kubectl apply -k`
- [ ] Create a simple CRD
- [ ] Install an operator (e.g., cert-manager)

**Lab Session 3 (16:30 - 19:00): Advanced Storage**
- [ ] Create StorageClass
- [ ] Enable dynamic provisioning
- [ ] Test access modes (RWO, ROX, RWX)
- [ ] Configure reclaim policies
- [ ] Expand PVCs
- [ ] Practice storage troubleshooting

#### ✅ Day 3 Checkpoints
- [ ] Can install/upgrade applications with Helm
- [ ] Can use Kustomize for multi-environment configs
- [ ] Understand CRD/Operator concepts
- [ ] Can configure dynamic storage provisioning

---

## 📅 DAY 4 - December 28 (Saturday)
### Focus: Services & Networking - Core Concepts

#### 🎯 Learning Objectives
- [ ] Master Service types and their use cases
- [ ] Understand Pod networking
- [ ] Implement and debug Network Policies
- [ ] Configure and troubleshoot CoreDNS

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **Pod Networking Model** (1 hour)
   - Pod-to-Pod communication
   - CNI plugins
   - IP address allocation
   - Read: [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

2. **Services Deep Dive** (1.5 hours)
   - ClusterIP, NodePort, LoadBalancer, ExternalName
   - Endpoints and EndpointSlices
   - Service discovery
   - Read: [Service](https://kubernetes.io/docs/concepts/services-networking/service/)

**Evening (20:00 - 21:30)**
3. **Network Policies** (1.5 hours)
   - Ingress and Egress rules
   - Selectors and namespaceSelectors
   - Default deny policies
   - Read: [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

4. **CoreDNS** (0.5 hours)
   - DNS for Services and Pods
   - CoreDNS configuration
   - Read: [DNS for Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): Services**
- [ ] Create ClusterIP service (imperative and declarative)
- [ ] Create NodePort service
- [ ] Test service discovery from within pods
- [ ] Examine endpoints
- [ ] Create headless services
- [ ] Debug service connectivity issues

**Lab Session 2 (13:30 - 16:00): Network Policies**
- [ ] Install network policy-supporting CNI (Calico)
- [ ] Create default deny policies
- [ ] Allow specific ingress traffic
- [ ] Allow specific egress traffic
- [ ] Test policies with different pods
- [ ] Namespace-based policies

**Lab Session 3 (16:30 - 19:00): CoreDNS**
- [ ] Examine CoreDNS configuration
- [ ] Test DNS resolution (service, pod)
- [ ] Debug DNS issues
- [ ] Customize CoreDNS ConfigMap
- [ ] Understand FQDN format
- [ ] External DNS resolution

#### ✅ Day 4 Checkpoints
- [ ] Can create all service types
- [ ] Can implement network policies
- [ ] Understand DNS naming conventions
- [ ] Can debug connectivity issues

---

## 📅 DAY 5 - December 29 (Sunday)
### Focus: Ingress, Gateway API + Workloads Basics

#### 🎯 Learning Objectives
- [ ] Configure Ingress controllers and resources
- [ ] Understand Gateway API basics
- [ ] Master Deployments and rolling updates
- [ ] Work with ConfigMaps and Secrets

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **Ingress** (1.5 hours)
   - Ingress controllers
   - Ingress resources and rules
   - TLS termination
   - Read: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

2. **Gateway API** (1 hour)
   - Gateway, GatewayClass, HTTPRoute
   - Differences from Ingress
   - Read: [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)

**Evening (20:00 - 21:30)**
3. **Deployments & ReplicaSets** (1.5 hours)
   - Deployment strategies
   - Rolling updates and rollbacks
   - Read: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

4. **ConfigMaps & Secrets** (1 hour)
   - Creating and using ConfigMaps
   - Creating and using Secrets
   - Environment variables vs volume mounts

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): Ingress**
- [ ] Install NGINX Ingress controller
- [ ] Create Ingress resource with path-based routing
- [ ] Create Ingress with host-based routing
- [ ] Configure TLS
- [ ] Debug Ingress issues
- [ ] Annotations for customization

**Lab Session 2 (13:30 - 16:00): Gateway API**
- [ ] Install Gateway API CRDs
- [ ] Create GatewayClass and Gateway
- [ ] Create HTTPRoute
- [ ] Route traffic to multiple services
- [ ] Compare with Ingress approach

**Lab Session 3 (16:30 - 19:00): Deployments & ConfigMaps**
- [ ] Create Deployments (imperative & declarative)
- [ ] Perform rolling updates
- [ ] Rollback to previous revision
- [ ] Create and use ConfigMaps
- [ ] Create and use Secrets
- [ ] Mount as volumes and env vars

#### ✅ Day 5 Checkpoints
- [ ] Can configure Ingress with TLS
- [ ] Understand Gateway API concepts
- [ ] Can perform rolling updates/rollbacks
- [ ] Can use ConfigMaps and Secrets

---

## 📅 DAY 6 - December 30 (Monday)
### Focus: Workloads Advanced + Troubleshooting Basics

#### 🎯 Learning Objectives
- [ ] Master Pod scheduling techniques
- [ ] Configure autoscaling
- [ ] Start systematic troubleshooting approach
- [ ] Debug node and pod issues

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **Advanced Scheduling** (1.5 hours)
   - Node selectors and affinity
   - Taints and tolerations
   - Pod anti-affinity
   - Read: [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

2. **Resource Management & Autoscaling** (1 hour)
   - Resource requests and limits
   - LimitRanges and ResourceQuotas
   - Horizontal Pod Autoscaler
   - Read: [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

**Evening (20:00 - 21:30)**
3. **Workload Controllers** (1 hour)
   - DaemonSets
   - StatefulSets
   - Jobs and CronJobs
   - Read: [Workload Resources](https://kubernetes.io/docs/concepts/workloads/controllers/)

4. **Troubleshooting Methodology** (1.5 hours)
   - Systematic debugging approach
   - Logs and events
   - Read: [Troubleshooting](https://kubernetes.io/docs/tasks/debug/)

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): Scheduling**
- [ ] Use nodeSelector
- [ ] Configure node affinity (required/preferred)
- [ ] Apply taints to nodes
- [ ] Configure tolerations in pods
- [ ] Pod anti-affinity for high availability
- [ ] Resource requests and limits

**Lab Session 2 (13:30 - 16:00): Autoscaling & Controllers**
- [ ] Install metrics-server
- [ ] Create HPA
- [ ] Test autoscaling with load
- [ ] Create DaemonSet
- [ ] Create StatefulSet
- [ ] Create Jobs and CronJobs

**Lab Session 3 (16:30 - 19:00): Basic Troubleshooting**
- [ ] Debug pods: CrashLoopBackOff
- [ ] Debug pods: ImagePullBackOff
- [ ] Debug pods: Pending state
- [ ] Check container logs
- [ ] Use kubectl describe for events
- [ ] Debug node NotReady

#### ✅ Day 6 Checkpoints
- [ ] Can configure all scheduling options
- [ ] Can setup and test HPA
- [ ] Can create all workload types
- [ ] Have systematic debugging approach

---

## 📅 DAY 7 - December 31 (Tuesday)
### Focus: Troubleshooting Intensive

#### 🎯 Learning Objectives
- [ ] Master cluster component troubleshooting
- [ ] Debug networking issues
- [ ] Troubleshoot application issues
- [ ] Time management for troubleshooting questions

#### 📚 Theory Session (5.5 hours)

**Morning (06:00 - 09:00)**
1. **Control Plane Troubleshooting** (1.5 hours)
   - API Server issues
   - Scheduler problems
   - Controller Manager issues
   - etcd problems

2. **Node Troubleshooting** (1 hour)
   - kubelet issues
   - Container runtime problems
   - Node conditions

**Evening (20:00 - 21:30)**
3. **Networking Troubleshooting** (1.5 hours)
   - Service not reachable
   - DNS resolution failures
   - Network policy blocking
   - Ingress issues

4. **Application Troubleshooting** (1 hour)
   - Logs analysis
   - Resource constraints
   - Liveness/readiness probes

#### 🔧 Hands-on Labs (8.5 hours)

**Lab Session 1 (09:30 - 12:30): Cluster Troubleshooting**
- [ ] Break and fix API server
- [ ] Break and fix scheduler
- [ ] Break and fix controller manager
- [ ] Break and fix kubelet
- [ ] Debug certificate issues
- [ ] Fix kubeconfig problems

**Lab Session 2 (13:30 - 16:00): Network Troubleshooting**
- [ ] Debug service connectivity
- [ ] Fix DNS resolution issues
- [ ] Debug network policy problems
- [ ] Fix Ingress not working
- [ ] Debug pod-to-pod communication
- [ ] Troubleshoot kube-proxy

**Lab Session 3 (16:30 - 19:00): Mixed Scenarios**
- [ ] Complex multi-issue scenarios
- [ ] Timed troubleshooting challenges
- [ ] Practice with killer.sh style questions
- [ ] Document common patterns
- [ ] Build troubleshooting checklist

#### ✅ Day 7 Checkpoints
- [ ] Can fix control plane issues
- [ ] Can debug networking problems
- [ ] Have troubleshooting patterns memorized
- [ ] Can work under time pressure

---

## 📅 DAY 8 - January 1 (Wednesday)
### Focus: Mock Exams + Final Review

#### 🎯 Learning Objectives
- [ ] Complete full mock exams under time pressure
- [ ] Identify and fix weak areas
- [ ] Finalize exam strategy
- [ ] Prepare mentally for the exam

#### 📖 Day 8 Schedule

**Morning (06:00 - 12:30)**
1. **Mock Exam 1** (2 hours) - Timed, no breaks
2. **Review Answers** (1.5 hours) - Analyze mistakes
3. **Practice Weak Areas** (2 hours)

**Afternoon (13:30 - 19:00)**
1. **Mock Exam 2** (2 hours) - Different question set
2. **Review Answers** (1.5 hours)
3. **Final Weak Area Practice** (2 hours)

**Evening (20:00 - 22:00)**
1. **Quick Review of All Topics** (1 hour)
2. **Exam Prep Checklist** - ID ready, environment ready
3. **Mental Preparation** - Early sleep, calm mind

#### Mock Exam Sources
1. **Killer.sh** - 2 free sessions included with CKA registration
2. **KodeKloud CKA Mock Exams**
3. **Practice scenarios from this guide**

---

## Resources & Documentation

### Official Kubernetes Documentation (Exam Allowed!)
- [Kubernetes.io Docs](https://kubernetes.io/docs/) - Main reference
- [Kubernetes.io Blog](https://kubernetes.io/blog/) - Updates and announcements
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### Key Documentation Pages to Bookmark
1. [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
2. [kubeadm Reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
3. [API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
4. [Well-Known Labels](https://kubernetes.io/docs/reference/labels-annotations-taints/)

### Practice Platforms
1. **Killer.sh** - Most exam-like experience
2. **KodeKloud** - Excellent labs
3. **Play with Kubernetes** - Quick sandbox

---

## 💡 Exam Day Tips

1. **Time Management**
   - 2 hours for ~17 questions
   - Average 7 minutes per question
   - Flag difficult questions and return

2. **Use Imperative Commands**
   - `kubectl run`, `kubectl create`, `kubectl expose`
   - Faster than writing YAML

3. **Use Documentation Efficiently**
   - Ctrl+F to search
   - Copy-paste YAML templates
   - Pre-bookmark key pages

4. **Terminal Setup**
   - `alias k=kubectl`
   - Enable kubectl autocompletion
   - Use `export do="--dry-run=client -o yaml"`

5. **Read Questions Carefully**
   - Note the namespace
   - Note the context/cluster
   - Check what's being asked

---

**Good luck with your CKA exam! 🚀**
