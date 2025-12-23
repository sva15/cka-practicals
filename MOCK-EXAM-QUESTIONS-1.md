# CKA Mock Exam Questions - Part 1

> **Difficulty Level:** Hard (Exam-like)  
> **Questions:** 1-40  
> **No Solutions Provided** - Practice on your cluster!  
> **See also:** [MOCK-EXAM-QUESTIONS-2.md](./MOCK-EXAM-QUESTIONS-2.md) for Questions 41-80  
> **See also:** [MOCK-EXAM-QUESTIONS-3.md](./MOCK-EXAM-QUESTIONS-3.md) for Questions 81-110

---

## Instructions

- Each question simulates real CKA exam format
- Context and namespace are specified per question
- Time yourself: aim for 5-7 minutes per question
- Verify your work after each question
- Questions are weighted by exam domain percentages

---

## Cluster Architecture, Installation & Configuration (25%)

### Question 1 - RBAC
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `development`

Create a ServiceAccount named `cicd-sa` in the `development` namespace. Create a Role named `deployment-manager` that allows:
- get, list, watch, create, update, delete on deployments
- get, list on pods
- get, list on services

Bind this role to the ServiceAccount. Verify the ServiceAccount can create deployments but cannot delete pods.

---

### Question 2 - RBAC Debugging
**Context:** `kubectl config use-context k8s-cluster1`

A user named `developer1` reports they cannot create pods in the `staging` namespace but should be able to. Investigate the current RBAC configuration for this user and fix the issue so they can create, get, list, and delete pods in `staging` namespace only.

---

### Question 3 - ClusterRole
**Context:** `kubectl config use-context k8s-cluster1`

Create a ClusterRole named `secret-reader` that allows reading secrets across all namespaces. Create a ClusterRoleBinding to bind this role to a ServiceAccount named `monitoring-sa` in the `monitoring` namespace. The ServiceAccount should be created if it doesn't exist.

---

### Question 4 - kubeadm Cluster Upgrade
**Context:** `kubectl config use-context k8s-cluster2`

The cluster is currently running Kubernetes v1.28.0. Upgrade the control plane node `master01` to v1.29.0. Ensure you:
- Drain the node appropriately
- Upgrade kubeadm, kubelet, and kubectl
- Apply the upgrade
- Uncordon the node

Document the exact commands you would use.

---

### Question 5 - kubeadm Worker Node
**Context:** `kubectl config use-context k8s-cluster2`

A new worker node `worker03` needs to be added to the cluster. Generate a new join token and the full join command. The token should be valid for 4 hours.

---

### Question 6 - etcd Backup
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

Take a snapshot of etcd and save it to `/opt/backup/etcd-snapshot-$(date +%Y%m%d).db`. The etcd is running as a stacked control plane component. Verify the snapshot was created successfully and show its status.

---

### Question 7 - etcd Restore
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

The etcd cluster has been corrupted. Restore it from the backup located at `/opt/backup/etcd-backup.db`. Ensure the cluster is fully operational after the restore.

---

### Question 8 - Certificate Check
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

Check the expiration date of all Kubernetes certificates on the control plane. Identify any certificates that will expire within the next 30 days and renew them.

---

### Question 9 - Helm Installation
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `database`

Using Helm, install MySQL from the Bitnami repository with:
- Release name: `mysql-primary`
- Namespace: `database` (create if not exists)
- Root password: `exampassword`
- Database name: `appdb`
- Persistence size: 5Gi

---

### Question 10 - Helm Upgrade and Rollback
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `database`

Upgrade the `mysql-primary` release to increase replicas to 2. Then, simulate a failed upgrade by changing to a non-existent image. Finally, rollback to the last working revision.

---

### Question 11 - Kustomize Base
**Context:** `kubectl config use-context k8s-cluster1`

Create a Kustomize configuration in `/opt/kustomize/base/` that includes:
- A deployment named `webapp` with nginx image and 2 replicas
- A ClusterIP service named `webapp-svc` on port 80
- Common label: `tier=frontend`

Apply the configuration to verify it works.

---

### Question 12 - Kustomize Overlay
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `production`

Using the base from Question 11, create a production overlay in `/opt/kustomize/overlays/production/` that:
- Changes replicas to 5
- Adds a namespace prefix `prod-`
- Sets namespace to `production`
- Adds resource limits: 256Mi memory, 200m CPU

---

### Question 13 - CRD Creation
**Context:** `kubectl config use-context k8s-cluster1`

Create a Custom Resource Definition named `databases.stable.example.com` with:
- Group: `stable.example.com`
- Kind: `Database`
- Plural: `databases`
- Scope: Namespaced
- Required fields: `engine` (string), `version` (string), `size` (integer)

Create a custom resource of this type named `mysql-prod`.

---

### Question 14 - Static Pod
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh worker01`

Create a static pod on `worker01` named `static-nginx` using the nginx:alpine image. The pod should:
- Listen on port 80
- Mount the node's /var/log as a volume at /var/log/host
- Have resource limits of 128Mi memory and 100m CPU

Verify the pod is running and managed by kubelet.

---

---

## Workloads & Scheduling (15%)

### Question 15 - Deployment Rolling Update
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `apps`

Create a deployment named `web-frontend` with:
- Image: `nginx:1.20`
- Replicas: 4
- Strategy: RollingUpdate with maxSurge=1, maxUnavailable=0
- Resource requests: 64Mi memory, 50m CPU

Perform a rolling update to nginx:1.23 and monitor the rollout. Then roll back to the previous version.

---

### Question 16 - Deployment with Probes
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `apps`

Create a deployment named `api-backend` with 3 replicas using `httpd:2.4` image that includes:
- Liveness probe: HTTP GET on "/" port 80, check every 10s, fail after 3 attempts
- Readiness probe: HTTP GET on "/" port 80, initial delay 5s
- Startup probe: HTTP GET on "/" port 80, failure threshold 30

---

### Question 17 - ConfigMap from Multiple Sources
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `config-test`

Create a ConfigMap named `app-settings` that contains:
- Key `LOG_LEVEL` with value `debug`
- Key `MAX_CONNECTIONS` with value `100`
- Contents of file `/opt/configs/database.properties`
- All files from directory `/opt/configs/extra/`

Create a pod that mounts this ConfigMap at `/etc/app-config`.

---

### Question 18 - Secret Management
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `secrets-lab`

Create a Secret named `api-credentials` containing:
- `api_key`: `sk-prod-abc123xyz`
- `api_secret`: `supersecretvalue`

Create a pod named `api-consumer` that:
- Uses `api_key` as environment variable `API_KEY`
- Mounts `api_secret` as a file at `/secrets/api-secret` with permissions 0400

---

### Question 19 - HPA Configuration
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `autoscale`

Create a deployment named `load-handler` with 2 replicas, using nginx image with CPU request of 100m. Create a Horizontal Pod Autoscaler that:
- Minimum replicas: 2
- Maximum replicas: 10
- Target CPU utilization: 50%
- Scale-down stabilization: 300 seconds

---

### Question 20 - Node Affinity
**Context:** `kubectl config use-context k8s-cluster1`

Create a pod named `gpu-workload` with image `nvidia/cuda:latest` that:
- MUST be scheduled on nodes with label `gpu=nvidia`
- PREFERS nodes with label `zone=us-west-1a`
- Includes a toleration for taint `gpu=dedicated:NoSchedule`

First, label node `worker02` with `gpu=nvidia` and taint it appropriately.

---

### Question 21 - Pod Anti-Affinity
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `ha-apps`

Create a deployment named `cache-server` with 3 replicas using redis:alpine image that ensures:
- No two pods of this deployment run on the same node (hard requirement)
- Pods prefer to run on nodes in different availability zones (soft requirement)

---

### Question 22 - Taints and Tolerations
**Context:** `kubectl config use-context k8s-cluster1`

Node `worker03` should be dedicated for database workloads only. Apply appropriate taints to achieve this. Then create a deployment named `db-cluster` with 2 replicas using postgres:14 image that can tolerate this taint and is forced to schedule on `worker03`.

---

### Question 23 - DaemonSet
**Context:** `kubectl config use-context k8s-cluster1`

Create a DaemonSet named `node-exporter` using `prom/node-exporter:latest` that:
- Runs on ALL nodes including control plane
- Mounts host's `/proc` at `/host/proc` (read-only)
- Mounts host's `/sys` at `/host/sys` (read-only)
- Uses host network
- Has resource limits of 100m CPU and 128Mi memory

---

### Question 24 - Job with Parallelism
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `batch`

Create a Job named `parallel-processor` that:
- Uses image `busybox`
- Runs command: `sh -c "echo Processing item $JOB_COMPLETION_INDEX && sleep 30"`
- Completes 10 times total
- Runs 3 pods in parallel
- Has backoff limit of 2
- Times out after 5 minutes

---

### Question 25 - CronJob
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `scheduled`

Create a CronJob named `db-backup` that:
- Runs every day at 2:30 AM
- Uses image `alpine`
- Executes: `echo "Backup started at $(date)" >> /backup/log.txt`
- Keeps last 5 successful job histories
- Keeps last 2 failed job histories
- Prevents concurrent runs

---

---

## Services & Networking (20%)

### Question 26 - Multi-Port Service
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `networking`

Create a deployment named `multi-app` with nginx image. Create a service that exposes:
- Port 80 (HTTP) targeting port 80
- Port 443 (HTTPS) targeting port 443
- Port 8080 (metrics) targeting port 8080

The service should be of type NodePort with specific ports: 30080, 30443, and 30808 respectively.

---

### Question 27 - Headless Service with StatefulSet
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `stateful`

Create a headless service named `mongo-svc` and a StatefulSet named `mongodb` with 3 replicas using mongo:5 image. Each pod should:
- Have stable network identity
- Use a volumeClaimTemplate for 1Gi persistent storage
- Be accessible via predictable DNS names

---

### Question 28 - Network Policy Default Deny
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `secure-ns`

Create a network policy that:
- Applies to all pods in the `secure-ns` namespace
- Denies all ingress traffic by default
- Denies all egress traffic by default

Verify pods in this namespace cannot communicate with each other or external services.

---

### Question 29 - Network Policy Allow Specific
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `secure-ns`

Building on Question 28, create an additional network policy that allows:
- Ingress to pods with label `role=api` only from pods with label `role=frontend`
- Ingress only on port 8080
- Egress from `role=api` pods to `role=database` pods on port 5432
- Egress to DNS (port 53 UDP) for all pods

---

### Question 30 - Network Policy Namespace Selector
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `backend`

Create a network policy that allows ingress traffic to pods in the `backend` namespace only from:
- Pods in the `frontend` namespace (any pod)
- Pods in the same namespace with label `app=internal-tool`
- The monitoring namespace on port 9090 only

Block all other ingress traffic.

---

### Question 31 - Ingress with Path-Based Routing
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `web-apps`

Create an Ingress resource named `app-ingress` that routes:
- `/api/*` to service `api-svc` on port 8080
- `/admin/*` to service `admin-svc` on port 80
- `/` (default) to service `frontend-svc` on port 80

Use the nginx ingress class and enable SSL redirect.

---

### Question 32 - Ingress with TLS
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `secure-web`

Create a TLS secret named `web-tls` using the certificate at `/opt/certs/tls.crt` and key at `/opt/certs/tls.key`. Create an Ingress named `secure-ingress` for host `secure.example.com` that:
- Uses the TLS secret
- Routes to service `secure-app` on port 443
- Forces HTTPS redirect

---

### Question 33 - CoreDNS Customization
**Context:** `kubectl config use-context k8s-cluster1`

Modify the CoreDNS configuration to:
- Add a custom hosts entry: `10.0.0.50 legacy-app.internal`
- Forward queries for `corp.example.com` to DNS server `172.16.0.53`
- Increase cache TTL to 60 seconds

Verify the changes work from within a pod.

---

### Question 34 - Service Troubleshooting
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `debug-ns`

There is a deployment named `web-app` and a service named `web-svc` in namespace `debug-ns`. Users report the service is not accessible. Diagnose the issue by checking:
- Service selector vs pod labels
- Endpoints
- Target port configuration
- Pod readiness

Fix any issues found.

---

### Question 35 - External Service
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `external`

Create an ExternalName service named `external-api` that points to `api.external-provider.com`. Create another service named `external-db` of type ClusterIP with manually configured endpoints pointing to:
- 192.168.1.100:5432
- 192.168.1.101:5432

---

---

## Troubleshooting (30%)

### Question 36 - Node NotReady
**Context:** `kubectl config use-context k8s-cluster3`

Node `worker02` is showing NotReady status. SSH to the node and diagnose the issue. Common causes to check:
- Kubelet status
- Container runtime status
- Certificate issues
- Network connectivity

Fix the issue and verify the node becomes Ready.

---

### Question 37 - Control Plane Component Failure
**Context:** `kubectl config use-context k8s-cluster3`  
**SSH:** `ssh master01`

The API server is not responding. Investigate the issue by checking:
- Static pod manifest
- Container logs
- Certificate configurations
- Required ports

Fix the issue and restore cluster functionality.

---

### Question 38 - Scheduler Failure
**Context:** `kubectl config use-context k8s-cluster3`

New pods are stuck in Pending state with no events. Investigate the scheduler:
- Check if kube-scheduler is running
- Examine scheduler logs
- Verify scheduler configuration

Fix any issues and verify pods can be scheduled.

---

### Question 39 - Pod CrashLoopBackOff
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `troubleshoot`

A pod named `failing-app` is in CrashLoopBackOff state. Diagnose why:
- Check container logs (current and previous)
- Examine resource limits
- Check liveness probe configuration
- Verify image and command

Document the root cause and fix.

---

### Question 40 - Pod ImagePullBackOff
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `troubleshoot`

A deployment named `private-app` has pods stuck in ImagePullBackOff. The image is hosted on a private registry `registry.example.com/apps/myapp:v2`. Diagnose and fix:
- Check if imagePullSecrets is configured
- Create necessary secrets
- Update the deployment

---

*Continue to [MOCK-EXAM-QUESTIONS-2.md](./MOCK-EXAM-QUESTIONS-2.md) for Questions 41-80*
