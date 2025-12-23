# CKA Workloads & Scheduling Scenarios (15% of Exam)

> **Exam Weight:** 15%  
> **Study Time:** ~17 hours across Days 4-6  
> **Total Scenarios:** 22

---

## Table of Contents
1. [Deployments & Rolling Updates (Scenarios 1-6)](#deployments--rolling-updates)
2. [ConfigMaps & Secrets (Scenarios 7-11)](#configmaps--secrets)
3. [Autoscaling (Scenarios 12-14)](#autoscaling)
4. [Workload Controllers (Scenarios 15-18)](#workload-controllers)
5. [Pod Scheduling (Scenarios 19-22)](#pod-scheduling)

---

## Deployments & Rolling Updates

### Scenario 1: Create a Deployment

**Objective:** Create a deployment using imperative and declarative methods.

**Requirements:**
- Name: `webapp`
- Image: `nginx:1.21`
- Replicas: 3

**Your Task:**
```bash
# Create the deployment both ways
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative method
kubectl create deployment webapp --image=nginx:1.21 --replicas=3

# Declarative method
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
EOF

# Verify
kubectl get deployment webapp
kubectl get pods -l app=webapp
```

</details>

---

### Scenario 2: Perform Rolling Update

**Objective:** Update deployment image with zero downtime.

**Requirements:**
- Update `webapp` from nginx:1.21 to nginx:1.23
- Monitor the rollout
- Verify all pods updated

**Your Task:**
```bash
# Update the deployment image
```

<details>
<summary>💡 Solution</summary>

```bash
# Method 1: kubectl set image
kubectl set image deployment/webapp nginx=nginx:1.23

# Method 2: kubectl edit
kubectl edit deployment webapp
# Change image to nginx:1.23

# Method 3: Patch
kubectl patch deployment webapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.23"}]}}}}'

# Monitor rollout
kubectl rollout status deployment/webapp

# Verify
kubectl describe deployment webapp | grep Image
kubectl get pods -l app=webapp -o jsonpath='{.items[*].spec.containers[0].image}'
```

</details>

---

### Scenario 3: Rollback a Deployment

**Objective:** Rollback to a previous revision.

**Requirements:**
- Update webapp to a broken image
- Rollback to previous working version
- Verify rollback

**Your Task:**
```bash
# Create a rollback scenario and fix it
```

<details>
<summary>💡 Solution</summary>

```bash
# Introduce a broken update
kubectl set image deployment/webapp nginx=nginx:broken

# Check rollout status
kubectl rollout status deployment/webapp
# Waiting... (pods failing)

# Check pod status
kubectl get pods -l app=webapp
# ImagePullBackOff

# View rollout history
kubectl rollout history deployment/webapp

# Rollback to previous
kubectl rollout undo deployment/webapp

# Or rollback to specific revision
kubectl rollout undo deployment/webapp --to-revision=1

# Verify
kubectl rollout status deployment/webapp
kubectl describe deployment webapp | grep Image
```

</details>

---

### Scenario 4: Configure Rolling Update Strategy

**Objective:** Customize the rolling update parameters.

**Requirements:**
- Set maxSurge: 1
- Set maxUnavailable: 0
- Verify controlled rollout

**Your Task:**
```bash
# Configure the strategy
```

<details>
<summary>💡 Solution</summary>

```yaml
# webapp-strategy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-controlled
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Max extra pods during update
      maxUnavailable: 0   # No pods can be unavailable
  selector:
    matchLabels:
      app: webapp-controlled
  template:
    metadata:
      labels:
        app: webapp-controlled
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

```bash
kubectl apply -f webapp-strategy.yaml

# Watch the update process
kubectl set image deployment/webapp-controlled nginx=nginx:1.23

# In another terminal
kubectl get pods -l app=webapp-controlled -w
# You'll see only 1 new pod at a time, no unavailable pods
```

**Strategy Types:**
- `RollingUpdate` (default): Gradual update
- `Recreate`: Kill all old pods, then create new

</details>

---

### Scenario 5: Scale a Deployment

**Objective:** Scale deployment replicas up and down.

**Requirements:**
- Scale webapp to 5 replicas
- Scale down to 2 replicas

**Your Task:**
```bash
# Scale the deployment
```

<details>
<summary>💡 Solution</summary>

```bash
# Scale up
kubectl scale deployment webapp --replicas=5

# Verify
kubectl get deployment webapp
kubectl get pods -l app=webapp

# Scale down
kubectl scale deployment webapp --replicas=2

# Alternative: Edit directly
kubectl edit deployment webapp
# Change replicas field

# Verify
kubectl get deployment webapp
```

</details>

---

### Scenario 6: Pause and Resume Rollout

**Objective:** Control rollout progress for batched updates.

**Your Task:**
```bash
# Pause a rollout, make changes, then resume
```

<details>
<summary>💡 Solution</summary>

```bash
# Pause the deployment
kubectl rollout pause deployment/webapp

# Make multiple changes (won't trigger rollout)
kubectl set image deployment/webapp nginx=nginx:1.24
kubectl set resources deployment/webapp -c nginx --requests=cpu=100m,memory=128Mi

# Resume to apply all changes at once
kubectl rollout resume deployment/webapp

# Monitor
kubectl rollout status deployment/webapp
```

</details>

---

## ConfigMaps & Secrets

### Scenario 7: Create ConfigMap (Multiple Methods)

**Objective:** Create ConfigMaps using different methods.

**Your Task:**
```bash
# Create ConfigMaps using literal, file, and directory
```

<details>
<summary>💡 Solution</summary>

```bash
# Method 1: From literals
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql.default.svc \
  --from-literal=DB_PORT=3306

# Method 2: From file
echo "log_level=debug" > app.properties
echo "max_connections=100" >> app.properties
kubectl create configmap app-file-config --from-file=app.properties

# Method 3: From directory
mkdir config-dir
echo "setting1=value1" > config-dir/file1
echo "setting2=value2" > config-dir/file2
kubectl create configmap app-dir-config --from-file=config-dir/

# Verify
kubectl get configmap app-config -o yaml
kubectl describe configmap app-file-config
```

</details>

---

### Scenario 8: Use ConfigMap as Environment Variables

**Objective:** Inject ConfigMap data as environment variables.

**Your Task:**
```bash
# Create a pod using ConfigMap as env vars
```

<details>
<summary>💡 Solution</summary>

```yaml
# configmap-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "DB_HOST=$DB_HOST DB_PORT=$DB_PORT" && sleep 3600']
    env:
    # Single variable from ConfigMap
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    # Or all variables from ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
```

```bash
kubectl apply -f configmap-env-pod.yaml
kubectl logs configmap-env-pod
# Output: DB_HOST=mysql.default.svc DB_PORT=3306
```

</details>

---

### Scenario 9: Mount ConfigMap as Volume

**Objective:** Mount ConfigMap as files in a pod.

**Your Task:**
```bash
# Mount ConfigMap as a volume
```

<details>
<summary>💡 Solution</summary>

```yaml
# configmap-volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'cat /config/DB_HOST && sleep 3600']
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # Optional: mount specific keys
      items:
      - key: DB_HOST
        path: database-host  # Custom filename
```

```bash
kubectl apply -f configmap-volume-pod.yaml
kubectl exec configmap-volume-pod -- ls /config
kubectl exec configmap-volume-pod -- cat /config/DB_HOST
```

</details>

---

### Scenario 10: Create and Use Secrets

**Objective:** Create secrets and use them in pods.

**Your Task:**
```bash
# Create secrets and mount them
```

<details>
<summary>💡 Solution</summary>

```bash
# Create secret from literals
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create secret from file
echo -n "my-api-key" > api-key.txt
kubectl create secret generic api-secret --from-file=api-key.txt

# Verify (base64 encoded)
kubectl get secret db-credentials -o yaml
```

```yaml
# secret-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "User: $DB_USER Pass: $DB_PASS" && sleep 3600']
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: secret-volume
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400
```

```bash
kubectl apply -f secret-pod.yaml
kubectl exec secret-pod -- cat /secrets/username
```

</details>

---

### Scenario 11: Update ConfigMap and See Changes

**Objective:** Understand how ConfigMap updates affect pods.

**Your Task:**
```bash
# Update ConfigMap and verify pod sees changes
```

<details>
<summary>💡 Solution</summary>

```bash
# Create ConfigMap and Pod with volume mount
kubectl create configmap live-config --from-literal=message=original

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: live-config-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do cat /config/message; sleep 5; done']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: live-config
EOF

# Watch logs
kubectl logs live-config-pod -f &

# Update ConfigMap
kubectl patch configmap live-config -p '{"data":{"message":"updated"}}'

# Wait ~30-60 seconds - volume-mounted ConfigMaps auto-update
# Note: env vars do NOT auto-update - pod restart required
```

</details>

---

## Autoscaling

### Scenario 12: Configure Horizontal Pod Autoscaler

**Objective:** Set up HPA to autoscale based on CPU.

**Requirements:**
- Target: webapp deployment
- Min replicas: 2
- Max replicas: 10
- Target CPU: 50%

**Your Task:**
```bash
# Create HPA for the deployment
```

<details>
<summary>💡 Solution</summary>

```bash
# Ensure metrics-server is installed
kubectl get deployment metrics-server -n kube-system

# If not installed:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Ensure deployment has resource requests
kubectl set resources deployment webapp --requests=cpu=50m

# Create HPA - imperative
kubectl autoscale deployment webapp --min=2 --max=10 --cpu-percent=50

# Or declarative
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

# Verify
kubectl get hpa webapp
```

</details>

---

### Scenario 13: Test HPA with Load

**Objective:** Generate load and watch HPA scale.

**Your Task:**
```bash
# Generate load and observe scaling
```

<details>
<summary>💡 Solution</summary>

```bash
# Watch HPA in one terminal
kubectl get hpa webapp -w

# Generate load in another terminal
kubectl run load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://webapp; done"

# Watch pods scale up
kubectl get pods -l app=webapp -w

# Check HPA status
kubectl describe hpa webapp

# Stop load
kubectl delete pod load-generator

# Watch scale down (takes a few minutes)
kubectl get hpa webapp -w
```

</details>

---

### Scenario 14: Configure Resource Requests and Limits

**Objective:** Set resource requests and limits on a deployment.

**Your Task:**
```bash
# Configure memory and CPU resources
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative
kubectl set resources deployment webapp \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi

# Verify
kubectl describe deployment webapp | grep -A5 Requests

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-resources
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-resources
  template:
    metadata:
      labels:
        app: webapp-resources
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
EOF
```

**Key Concepts:**
- Requests: Minimum guaranteed resources
- Limits: Maximum allowed resources
- Pod QoS Classes: Guaranteed (requests=limits), Burstable, BestEffort

</details>

---

## Workload Controllers

### Scenario 15: Create a DaemonSet

**Objective:** Deploy a pod on every node.

**Requirements:**
- Name: `log-collector`
- Image: `fluentd`
- Run on all nodes

**Your Task:**
```bash
# Create a DaemonSet
```

<details>
<summary>💡 Solution</summary>

```yaml
# log-collector-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluentd
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
      tolerations:
      # Run on control plane nodes too
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```

```bash
kubectl apply -f log-collector-ds.yaml

# Verify - one pod per node
kubectl get pods -l app=log-collector -o wide
kubectl get daemonset log-collector
```

</details>

---

### Scenario 16: Create a StatefulSet

**Objective:** Deploy stateful application with stable identities.

**Requirements:**
- Name: `web-sts`
- Image: `nginx`
- Replicas: 3
- Headless service

**Your Task:**
```bash
# Create StatefulSet with headless service
```

<details>
<summary>💡 Solution</summary>

```yaml
# statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None  # Headless service
  selector:
    app: web-sts
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts
spec:
  serviceName: "web-headless"
  replicas: 3
  selector:
    matchLabels:
      app: web-sts
  template:
    metadata:
      labels:
        app: web-sts
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f statefulset.yaml

# Pods have stable names: web-sts-0, web-sts-1, web-sts-2
kubectl get pods -l app=web-sts

# Each pod has stable DNS:
# web-sts-0.web-headless.default.svc.cluster.local
kubectl exec web-sts-0 -- hostname
```

</details>

---

### Scenario 17: Create a Job

**Objective:** Run a batch task to completion.

**Requirements:**
- Name: `batch-job`
- Image: `busybox`
- Task: Calculate pi to 2000 digits

**Your Task:**
```bash
# Create a Job
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative
kubectl create job batch-job --image=busybox -- \
  sh -c "echo 'scale=2000; 4*a(1)' | bc -l"

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  completions: 1           # How many times to complete
  parallelism: 1           # How many pods run in parallel
  backoffLimit: 4          # Retry count on failure
  activeDeadlineSeconds: 100  # Max time
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
EOF

# Check status
kubectl get jobs
kubectl get pods -l job-name=pi-calculator
kubectl logs job/pi-calculator
```

</details>

---

### Scenario 18: Create a CronJob

**Objective:** Schedule a recurring task.

**Requirements:**
- Name: `backup-job`
- Schedule: Every 5 minutes
- Command: Echo timestamp

**Your Task:**
```bash
# Create a CronJob
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative
kubectl create cronjob backup-job --image=busybox \
  --schedule="*/5 * * * *" \
  -- sh -c "echo Backup at $(date)"

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid  # Don't run if previous still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command: ['sh', '-c', 'echo "Backup at $(date)"']
          restartPolicy: OnFailure
EOF

# Verify
kubectl get cronjobs
kubectl get jobs --watch

# Manually trigger
kubectl create job backup-manual --from=cronjob/backup-job
```

**Cron Schedule Format:** `minute hour day month weekday`

</details>

---

## Pod Scheduling

### Scenario 19: Use Node Selector

**Objective:** Schedule pod on specific node using labels.

**Your Task:**
```bash
# Schedule pod on a node with specific label
```

<details>
<summary>💡 Solution</summary>

```bash
# Label a node
kubectl label node worker-1 disktype=ssd

# Create pod with nodeSelector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
EOF

# Verify
kubectl get pod ssd-pod -o wide
# Should be on worker-1
```

</details>

---

### Scenario 20: Configure Node Affinity

**Objective:** Use node affinity for flexible scheduling.

**Your Task:**
```bash
# Configure required and preferred node affinity
```

<details>
<summary>💡 Solution</summary>

```yaml
# node-affinity-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      # MUST match
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      # PREFER to match
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f node-affinity-pod.yaml
kubectl get pod affinity-pod -o wide
```

**Operators:** In, NotIn, Exists, DoesNotExist, Gt, Lt

</details>

---

### Scenario 21: Configure Taints and Tolerations

**Objective:** Control pod scheduling using taints.

**Your Task:**
```bash
# Taint a node and allow specific pods to schedule
```

<details>
<summary>💡 Solution</summary>

```bash
# Taint a node
kubectl taint nodes worker-2 dedicated=database:NoSchedule

# Regular pods won't schedule on worker-2
kubectl run test-pod --image=nginx
kubectl get pod test-pod -o wide
# Not on worker-2

# Create pod with toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: secret
EOF

# db-pod CAN schedule on worker-2
kubectl get pod db-pod -o wide

# Remove taint
kubectl taint nodes worker-2 dedicated=database:NoSchedule-
```

**Taint Effects:**
- `NoSchedule`: New pods won't schedule
- `PreferNoSchedule`: Soft version
- `NoExecute`: Evict existing pods too

</details>

---

### Scenario 22: Configure Pod Anti-Affinity

**Objective:** Spread pods across nodes for HA.

**Your Task:**
```bash
# Ensure pods don't run on same node
```

<details>
<summary>💡 Solution</summary>

```yaml
# anti-affinity-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-spread
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-spread
  template:
    metadata:
      labels:
        app: web-spread
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-spread
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx
```

```bash
kubectl apply -f anti-affinity-deploy.yaml

# Pods should be on different nodes
kubectl get pods -l app=web-spread -o wide
```

</details>

---

## Quick Reference

### Imperative Commands

```bash
# Deployments
kubectl create deployment NAME --image=IMAGE --replicas=N
kubectl scale deployment NAME --replicas=N
kubectl set image deployment/NAME CONTAINER=IMAGE
kubectl rollout status/history/undo deployment/NAME

# ConfigMaps
kubectl create configmap NAME --from-literal=K=V --from-file=FILE

# Secrets
kubectl create secret generic NAME --from-literal=K=V

# Jobs/CronJobs
kubectl create job NAME --image=IMAGE -- COMMAND
kubectl create cronjob NAME --image=IMAGE --schedule="* * * * *" -- CMD

# Autoscaling
kubectl autoscale deployment NAME --min=N --max=N --cpu-percent=N
```

---

**Next:** Move to [04-CLUSTER-ADMIN-SCENARIOS.md](./04-CLUSTER-ADMIN-SCENARIOS.md)
