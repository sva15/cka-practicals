# CKA Mock Exam Questions - Part 2

> **Difficulty Level:** Hard (Exam-like)  
> **Questions:** 41-80  
> **No Solutions Provided** - Practice on your cluster!  
> **See also:** [MOCK-EXAM-QUESTIONS-1.md](./MOCK-EXAM-QUESTIONS-1.md) for Questions 1-40  
> **See also:** [MOCK-EXAM-QUESTIONS-3.md](./MOCK-EXAM-QUESTIONS-3.md) for Questions 81-110

---

## Troubleshooting (30%) - Continued

### Question 41 - Application Logs Analysis
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `production`

A multi-container pod named `web-app` has containers `nginx` and `sidecar-logger`. The application is returning 500 errors. Extract the last 100 lines of logs from both containers and identify which container is causing the issue. Save the relevant error logs to `/opt/logs/error-analysis.txt`.

---

### Question 42 - Node Resource Pressure
**Context:** `kubectl config use-context k8s-cluster3`

Node `worker01` is experiencing memory pressure and evicting pods. Investigate:
- Current memory usage on the node
- Which pods are consuming the most memory
- Eviction thresholds configured

Identify the pod causing the pressure and take appropriate action to stabilize the node.

---

### Question 43 - Kubelet Troubleshooting
**Context:** `kubectl config use-context k8s-cluster3`  
**SSH:** `ssh worker03`

The kubelet on `worker03` is not starting properly. Diagnose the issue by checking:
- Kubelet service status
- Kubelet configuration file
- Kubelet logs
- Required directories and permissions

Fix the issue and verify kubelet starts successfully.

---

### Question 44 - kube-proxy Issues
**Context:** `kubectl config use-context k8s-cluster3`

Services are not accessible from within the cluster. Pods can ping each other by IP but not via service names. Diagnose kube-proxy:
- Check kube-proxy pods
- Verify iptables/ipvs rules
- Check kube-proxy configuration

Fix any issues found.

---

### Question 45 - DNS Resolution Failure
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `dns-test`

Pods in the `dns-test` namespace cannot resolve service names. They can reach external IPs but not `kubernetes.default.svc.cluster.local`. Diagnose:
- CoreDNS pod status
- CoreDNS logs
- Pod DNS configuration
- CoreDNS service endpoints

Fix the DNS resolution issue.

---

### Question 46 - CNI Issues
**Context:** `kubectl config use-context k8s-cluster3`  
**SSH:** `ssh worker02`

Newly created pods on `worker02` are stuck in ContainerCreating state. The pod events show network-related errors. Diagnose:
- CNI plugin configuration
- CNI binary availability
- Network namespace creation

Fix the issue so pods can be created successfully.

---

### Question 47 - PVC Pending
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `storage-debug`

A PVC named `data-pvc` has been Pending for 10 minutes. Diagnose why:
- Check for matching PVs
- Verify StorageClass configuration
- Check provisioner logs if using dynamic provisioning

Fix the issue to get the PVC bound.

---

### Question 48 - etcd Performance
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

The cluster is experiencing slow API responses. Investigate etcd performance:
- Check etcd metrics
- Verify etcd cluster health
- Check disk latency
- Monitor etcd member status

Document findings and recommend fixes.

---

### Question 49 - Certificate Troubleshooting
**Context:** `kubectl config use-context k8s-cluster2`  
**SSH:** `ssh master01`

kubectl commands are failing with certificate errors. Diagnose:
- API server certificate validity
- kubeconfig certificate references
- Client certificate expiration

Fix the certificate issues to restore cluster access.

---

### Question 50 - Controller Manager Issues
**Context:** `kubectl config use-context k8s-cluster3`

Deployments are not creating ReplicaSets and pods are not being replaced when deleted. Investigate the controller manager:
- Check controller manager pod status
- Examine controller manager logs
- Verify leader election status

Fix the issue to restore deployment functionality.

---

---

## Storage (10%)

### Question 51 - PV with Multiple Access Modes
**Context:** `kubectl config use-context k8s-cluster1`

Create a PersistentVolume named `shared-pv` with:
- Capacity: 5Gi
- Access Modes: ReadWriteOnce and ReadOnlyMany
- hostPath: `/data/shared`
- Reclaim Policy: Retain
- Storage Class: `manual`

Create a PVC that binds to it with ReadWriteOnce mode.

---

### Question 52 - Dynamic Provisioning
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `dynamic-storage`

Create a StorageClass named `fast-storage` with:
- Provisioner: `kubernetes.io/no-provisioner`
- Volume binding mode: `WaitForFirstConsumer`
- Allow volume expansion: true

Create a PVC that uses this storage class and a pod that consumes it. Verify the binding happens only when the pod is scheduled.

---

### Question 53 - PV Reclaim
**Context:** `kubectl config use-context k8s-cluster1`

A PV named `data-pv` is in Released state after its PVC was deleted. The data on the PV needs to be preserved. Make this PV available for a new PVC to bind to while keeping the existing data intact.

---

### Question 54 - Expand PVC
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `storage-expand`

A PVC named `app-data` currently has 2Gi storage and is bound to a pod. The application needs more space. Expand the PVC to 5Gi without data loss. Verify the expansion is reflected inside the pod.

---

### Question 55 - Storage Class with Parameters
**Context:** `kubectl config use-context k8s-cluster1`

Create a StorageClass named `encrypted-storage` with:
- Appropriate provisioner for your environment
- Reclaim policy: Delete
- Volume binding mode: Immediate
- Parameters for encryption (if applicable to your provisioner)

---

### Question 56 - Volume Snapshot
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `snapshot-ns`

Assuming VolumeSnapshot CRDs are installed, create a VolumeSnapshot named `db-snapshot` from an existing PVC named `database-pvc`. Then create a new PVC named `restored-db-pvc` from this snapshot.

---

### Question 57 - NFS Volume
**Context:** `kubectl config use-context k8s-cluster1`

Create a PersistentVolume named `nfs-pv` that:
- Uses NFS storage at server `192.168.1.50` path `/exports/data`
- Has capacity 10Gi
- Supports ReadWriteMany access mode

Create a deployment with 3 replicas that all mount this volume at `/shared-data`.

---

### Question 58 - Projected Volume
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `projected-test`

Create a pod named `projected-pod` that uses a projected volume containing:
- ServiceAccount token
- ConfigMap named `app-config`
- Secret named `app-secret`
- Downward API for pod name and namespace

Mount all at `/etc/projected`.

---

### Question 59 - CSI Volume
**Context:** `kubectl config use-context k8s-cluster1`

List all CSI drivers installed in the cluster. Create a PVC using a CSI-based StorageClass and mount it in a pod. Verify the volume was provisioned by the CSI driver.

---

### Question 60 - Storage Troubleshooting
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `storage-issue`

A pod named `db-pod` cannot start because its volume mount is failing. The PVC appears bound but the pod shows volume-related errors. Diagnose:
- Mount permissions
- Volume availability on the node
- StorageClass provisioner status

Fix the issue.

---

---

## Workloads & Scheduling (15%) - Continued

### Question 61 - StatefulSet with Init Containers
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `stateful-app`

Create a StatefulSet named `mysql-cluster` with 3 replicas where:
- Init container downloads config from a URL
- Main container uses mysql:8 image
- Each pod has its own 5Gi persistent volume
- Pods are created sequentially
- Headless service named `mysql-headless`

---

### Question 62 - Pod Disruption Budget
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `pdb-test`

Create a deployment named `critical-app` with 5 replicas. Create a PodDisruptionBudget that ensures:
- Minimum 3 pods are always available
- Or alternatively, maximum 1 pod unavailable during voluntary disruptions

Test by attempting to drain a node.

---

### Question 63 - Priority Classes
**Context:** `kubectl config use-context k8s-cluster1`

Create a PriorityClass named `high-priority` with value 1000000. Create another named `low-priority` with value 1000. Create a pod using the high-priority class. Verify priority-based preemption works when resources are constrained.

---

### Question 64 - Resource Quotas
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `quota-ns`

Create a ResourceQuota in namespace `quota-ns` that limits:
- Maximum 10 pods
- Maximum 4 CPU total requests
- Maximum 8Gi memory total requests
- Maximum 8 CPU total limits
- Maximum 16Gi memory total limits
- Maximum 5 PVCs

Verify the quota by attempting to exceed it.

---

### Question 65 - LimitRange
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `limit-ns`

Create a LimitRange that:
- Sets default CPU request to 100m and limit to 500m
- Sets default memory request to 128Mi and limit to 512Mi
- Sets minimum CPU to 50m and maximum to 2
- Sets minimum memory to 64Mi and maximum to 2Gi

Create a pod without resource specifications and verify defaults are applied.

---

---

## Services & Networking (20%) - Continued

### Question 66 - Complex Network Policy
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `multi-tier`

Create network policies for a 3-tier application:
- `frontend` pods can only receive traffic from outside the cluster (ingress controller)
- `backend` pods can only receive traffic from `frontend` pods
- `database` pods can only receive traffic from `backend` pods on port 5432
- All tiers can make DNS queries
- No other traffic is allowed

---

### Question 67 - Egress Network Policy
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `egress-control`

Create a network policy that allows pods with label `app=external-client` to:
- Connect to external IP range 10.0.0.0/8 on port 443
- Connect to the Kubernetes API (port 443)
- Make DNS queries
- Block all other egress traffic

---

### Question 68 - Ingress with Multiple Hosts
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `multi-host`

Create a single Ingress resource that:
- Routes `app1.example.com` to service `app1-svc`
- Routes `app2.example.com` to service `app2-svc`
- Routes `api.example.com/v1/*` to service `api-v1-svc`
- Routes `api.example.com/v2/*` to service `api-v2-svc`
- Uses TLS for all hosts with appropriate secrets

---

### Question 69 - Service Mesh Preparation
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `mesh-prep`

Prepare a namespace for service mesh injection by:
- Creating the namespace with appropriate labels for automatic sidecar injection
- Creating a deployment that will receive sidecars
- Creating service-to-service communication that the mesh can intercept

(This tests understanding of how service meshes work with K8s primitives)

---

### Question 70 - Gateway API
**Context:** `kubectl config use-context k8s-cluster1`

Using Gateway API resources (if CRDs are installed):
- Create a Gateway named `main-gateway` listening on port 80 and 443
- Create an HTTPRoute that routes based on headers
- Create an HTTPRoute with traffic splitting (80% to v1, 20% to v2)

---

### Question 71 - Service with ExternalIPs
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `external-access`

Create a Service named `external-web` that:
- Is of type ClusterIP
- Has an external IP of `192.168.100.50`
- Exposes port 80
- Selects pods with label `app=web`

Explain when this approach would be used vs LoadBalancer.

---

### Question 72 - Endpoint Slices
**Context:** `kubectl config use-context k8s-cluster1`

Examine the EndpointSlices for an existing service. Create a service with more than 100 endpoints and observe how EndpointSlices are managed. Document the difference between Endpoints and EndpointSlices.

---

---

## Cluster Architecture (25%) - Continued

### Question 73 - High Availability etcd
**Context:** `kubectl config use-context k8s-cluster2`

The cluster uses stacked etcd. Document the steps to add a new etcd member to the cluster. Include:
- Prerequisites for the new node
- Commands to add the member
- Verification steps
- How to handle failures during the process

---

### Question 74 - Control Plane HA
**Context:** `kubectl config use-context k8s-cluster2`

Add a second control plane node to the existing cluster. The new node is `master02`. Document:
- Certificate requirements
- Load balancer configuration
- kubeadm join process for control plane
- Verification of HA functionality

---

### Question 75 - Cluster Backup Strategy
**Context:** `kubectl config use-context k8s-cluster1`

Create a comprehensive backup strategy that includes:
- etcd backup to `/opt/backups/etcd/`
- All cluster resource definitions to `/opt/backups/resources/`
- Certificate backup to `/opt/backups/certs/`
- Create a script that can be run as a CronJob

---

### Question 76 - Node Maintenance
**Context:** `kubectl config use-context k8s-cluster1`

Node `worker02` requires kernel upgrade and reboot. Perform maintenance:
- Safely evict all workloads
- Prevent new pods from scheduling
- After maintenance, restore normal operation
- Verify all pods are rescheduled properly

---

### Question 77 - RBAC Audit
**Context:** `kubectl config use-context k8s-cluster1`

Perform a security audit of RBAC:
- List all ClusterRoleBindings that give cluster-admin access
- Find all ServiceAccounts with elevated privileges
- Identify any overly permissive Roles
- Document findings and recommendations

---

### Question 78 - API Server Flags
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

Modify the API server to:
- Enable audit logging to `/var/log/kubernetes/audit.log`
- Set audit policy from `/etc/kubernetes/audit-policy.yaml`
- Enable admission controller `PodSecurityPolicy`
- Increase `--max-requests-inflight` to 800

Verify changes take effect after restart.

---

### Question 79 - Scheduler Configuration
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

Create a custom scheduler configuration that:
- Uses a specific scoring plugin
- Modifies scheduler performance tuning parameters
- Logs at debug level

Deploy a pod that uses this custom scheduler configuration.

---

### Question 80 - Extension Resources
**Context:** `kubectl config use-context k8s-cluster1`

Install and configure an operator from OperatorHub. After installation:
- Verify the CRDs are created
- Create a custom resource instance
- Verify the operator reconciles the resource
- Check operator logs for any issues

---

*Continue to [MOCK-EXAM-QUESTIONS-3.md](./MOCK-EXAM-QUESTIONS-3.md) for Questions 81-110*
