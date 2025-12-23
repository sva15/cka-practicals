# CKA Mock Exam Questions - Part 3

> **Difficulty Level:** Hard (Exam-like)  
> **Questions:** 81-110  
> **No Solutions Provided** - Practice on your cluster!  
> **See also:** [MOCK-EXAM-QUESTIONS-1.md](./MOCK-EXAM-QUESTIONS-1.md) for Questions 1-40  
> **See also:** [MOCK-EXAM-QUESTIONS-2.md](./MOCK-EXAM-QUESTIONS-2.md) for Questions 41-80

---

## Mixed Domain Questions (Advanced)

These questions combine multiple concepts and require deeper understanding.

---

### Question 81 - End-to-End Deployment
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `e2e-app`

Deploy a complete 3-tier application:
1. Database: StatefulSet with postgres:14, 1 replica, 5Gi PVC, headless service
2. Backend: Deployment with 3 replicas, connects to DB via service name
3. Frontend: Deployment with 2 replicas, connects to backend
4. Expose frontend via Ingress at `app.example.com`
5. Create network policies: frontend→backend→database only
6. Configure HPA for backend (2-6 replicas, 70% CPU target)

---

### Question 82 - Cluster Disaster Recovery
**Context:** `kubectl config use-context k8s-cluster2`

Simulate and recover from a disaster:
1. Take etcd backup
2. Document all critical resources in `kube-system`
3. "Corrupt" etcd by deleting a critical object
4. Restore from backup
5. Verify cluster is fully operational

---

### Question 83 - Security Hardening
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `secure-app`

Deploy an application with maximum security:
- Pod runs as non-root user (UID 1000)
- Read-only root filesystem
- No privilege escalation
- Drops all capabilities except NET_BIND_SERVICE
- Uses a dedicated ServiceAccount with minimal permissions
- Network policy restricting traffic to only required communications

---

### Question 84 - Multi-Cluster Resource
**Context:** Multiple clusters

You have access to two clusters: `cluster-east` and `cluster-west`. Create a deployment in both clusters and configure:
- Same application running in both
- Document how you would set up cross-cluster service discovery
- Explain the networking requirements

---

### Question 85 - Canary Deployment
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `canary`

Implement a canary deployment:
- Create deployment `app-stable` with 9 replicas running `nginx:1.20`
- Create deployment `app-canary` with 1 replica running `nginx:1.23`
- Create a single service that routes to both (90% stable, 10% canary by replica count)
- Verify traffic distribution

---

### Question 86 - Blue-Green Deployment
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `bluegreen`

Implement blue-green deployment:
- Deploy `app-blue` with nginx:1.20
- Deploy `app-green` with nginx:1.23
- Create service pointing to blue
- Switch service to green
- Rollback to blue if issues

---

### Question 87 - Logging Stack
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `logging`

Deploy a logging solution:
- DaemonSet to collect logs from `/var/log/containers/`
- Verify logs are collected from all nodes
- Configure volume mounts for log access
- Set appropriate resource limits

---

### Question 88 - Monitoring Setup
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `monitoring`

Deploy metrics collection:
- Install metrics-server if not present
- Create ServiceMonitor (if Prometheus Operator exists) or equivalent
- Verify `kubectl top nodes` and `kubectl top pods` work
- Create HPA that uses these metrics

---

### Question 89 - Init Container Dependencies
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `init-demo`

Create a pod with multiple init containers in sequence:
1. First init: Waits for a ConfigMap to exist
2. Second init: Downloads a file from a URL
3. Third init: Validates the downloaded file
4. Main container: Uses the downloaded file

Each init container should pass data to the next via shared volume.

---

### Question 90 - Sidecar Pattern
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `sidecar-demo`

Implement sidecar pattern:
- Main container: Application writing logs to `/var/log/app/`
- Sidecar: Reads logs and forwards to stdout
- Shared emptyDir volume
- Appropriate lifecycle ordering

---

---

## Advanced Troubleshooting

### Question 91 - Complete Cluster Recovery
**Context:** `kubectl config use-context k8s-cluster3`

The cluster is in a completely broken state:
- API server certificate expired
- etcd has connectivity issues
- kubelet on one node not starting

Diagnose and fix ALL issues to restore full cluster functionality.

---

### Question 92 - Performance Debugging
**Context:** `kubectl config use-context k8s-cluster1`

Applications report slow response times. Investigate:
- API server latency
- etcd performance
- Node resource utilization
- Pod resource consumption
- Network latency between pods

Create a performance report with findings.

---

### Question 93 - Network Trace
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `trace-ns`

Pod `client` cannot reach pod `server` in the same namespace. Systematically trace:
- Pod IPs and network configuration
- iptables rules
- Network policy effects
- CNI plugin logs
- DNS resolution

Document each step of the investigation.

---

### Question 94 - Storage Corruption
**Context:** `kubectl config use-context k8s-cluster1`

A PV shows as Available but pods mounting it fail with I/O errors. Diagnose:
- PV/PVC binding status
- Underlying storage health
- Mount point accessibility on the node
- File system integrity

---

### Question 95 - Scheduling Failure Analysis
**Context:** `kubectl config use-context k8s-cluster1`

A pod has been Pending for 30 minutes. Perform complete analysis:
- Resource availability across nodes
- Node taints and tolerations
- Node affinity rules
- PVC binding status
- Pod priority vs other pods

Document root cause and resolution.

---

---

## Scenario-Based Questions

### Question 96 - Namespace Isolation
**Context:** `kubectl config use-context k8s-cluster1`

Create a completely isolated namespace `team-finance` where:
- Only members of `finance-team` group can access
- Pods can only communicate within the namespace
- Resource quota limits total resources
- LimitRange enforces per-pod limits
- NetworkPolicy blocks all external traffic

---

### Question 97 - Secrets Rotation
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `secrets-rotation`

A deployment `secure-app` uses Secret `db-credentials`. The secret needs to be rotated:
- Create new secret `db-credentials-v2`
- Update deployment to use new secret
- Verify pods restart with new credentials
- Delete old secret
- Document the zero-downtime approach

---

### Question 98 - ConfigMap Hot Reload
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `config-reload`

Create an application that:
- Uses ConfigMap mounted as volume
- Detects changes to ConfigMap automatically
- Reloads configuration without pod restart
- Logs when configuration changes

---

### Question 99 - External Database Connection
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `external-db`

Configure pods to connect to an external database at `192.168.1.100:5432`:
- Create Service and Endpoints manually
- Store credentials in Secret
- Create NetworkPolicy allowing this egress
- Test connectivity from a pod

---

### Question 100 - Service Account Token Projection
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `token-demo`

Create a pod that uses a projected ServiceAccount token:
- Token audience: `api.example.com`
- Token expiration: 1 hour
- Verify the token contains expected claims
- Show how to refresh the token

---

---

## Expert-Level Questions

### Question 101 - Custom Scheduler
**Context:** `kubectl config use-context k8s-cluster1`

Deploy a second scheduler named `my-scheduler`:
- Use the same scheduler image as default
- Configure with different leader-election namespace
- Create pods that use this custom scheduler
- Verify pods are scheduled by custom scheduler

---

### Question 102 - Admission Controller
**Context:** `kubectl config use-context k8s-cluster1`

Configure an admission webhook:
- Deploy a simple validating webhook server
- Create ValidatingWebhookConfiguration
- Webhook should reject pods without specific label
- Test that pods without label are rejected

---

### Question 103 - API Aggregation
**Context:** `kubectl config use-context k8s-cluster1`

Understand API aggregation:
- List all APIServices
- Identify which are served by aggregation layer
- Check health of aggregated APIs
- Understand how metrics-server uses aggregation

---

### Question 104 - Encryption at Rest
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

Configure etcd encryption:
- Create encryption configuration file
- Configure API server to use encryption
- Verify secrets are encrypted in etcd
- Rotate encryption keys

---

### Question 105 - Audit Logging
**Context:** `kubectl config use-context k8s-cluster1`  
**SSH:** `ssh master01`

Configure comprehensive audit logging:
- Create audit policy with multiple rules
- Log all writes to secrets
- Log metadata for all requests
- Exclude health checks from logging
- Verify audit logs are written

---

### Question 106 - Pod Security Standards
**Context:** `kubectl config use-context k8s-cluster1`

Configure namespace with Pod Security Standards:
- Create namespace `restricted-ns` with `restricted` security level
- Create namespace `baseline-ns` with `baseline` security level
- Attempt to create non-compliant pods
- Verify enforcement works correctly

---

### Question 107 - IPv6 Dual-Stack
**Context:** `kubectl config use-context k8s-cluster1`

If dual-stack is enabled:
- Create a service with both IPv4 and IPv6 addresses
- Create pods that can communicate over both protocols
- Configure appropriate network policies for IPv6

---

### Question 108 - Ephemeral Containers
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `debug`

A running pod `production-app` is experiencing issues but has no shell:
- Use ephemeral containers to debug
- Add a debug container with troubleshooting tools
- Investigate the issue from within the pod namespace
- Document the debugging process

---

### Question 109 - VPA (if installed)
**Context:** `kubectl config use-context k8s-cluster1`  
**Namespace:** `vpa-test`

If Vertical Pod Autoscaler is available:
- Create a deployment with minimal resources
- Configure VPA to recommend resources
- Set VPA to auto mode
- Observe resource adjustments

---

### Question 110 - Complete Environment Setup
**Context:** `kubectl config use-context k8s-cluster1`

Set up a complete development environment:
1. Namespace with appropriate quotas and limits
2. RBAC for developer group
3. Network policies for isolation
4. Default container registry secret
5. ConfigMaps for environment configuration
6. Service to expose internal tools
7. Ingress for external access
8. Documentation of the setup

---

---

## Question Summary by Domain

| Domain | Question Numbers | Count |
|--------|------------------|-------|
| Cluster Architecture | 1-14, 73-80, 101-107 | 29 |
| Workloads & Scheduling | 15-25, 61-65, 85-90 | 22 |
| Services & Networking | 26-35, 66-72, 99 | 18 |
| Troubleshooting | 36-50, 91-95 | 20 |
| Storage | 51-60, 94 | 11 |
| Mixed/Advanced | 81-84, 96-98, 100, 108-110 | 10 |
| **Total** | | **110** |

---

## Time Management Practice

| Difficulty | Questions | Time per Question |
|------------|-----------|-------------------|
| Standard | 1-50 | 5-6 minutes |
| Advanced | 51-80 | 6-8 minutes |
| Expert | 81-110 | 8-10 minutes |

**Exam Simulation:**
- Select 17-20 random questions
- Complete in 2 hours
- No solutions/documentation peek first attempt
- Documentation allowed on second attempt

---

## Verification Checklist

After each question, verify:
- [ ] Resources created with correct names
- [ ] Correct namespace used
- [ ] Labels and selectors match
- [ ] Pods are Running (not Pending/Error)
- [ ] Services have endpoints
- [ ] Network policies allow intended traffic
- [ ] RBAC permissions work as expected

---

**Good luck with your practice! 🎯**

*These 110 questions cover the entire December 2025 CKA syllabus with exam-level difficulty.*
