# CKA Cluster Admin Scenarios Part 2 (25% of Exam)

> **Part 2 Topics:** Helm, Kustomize, CRDs, HA, Extensions  
> **This File:** 14 scenarios  
> **See Also:** [04A-CLUSTER-ADMIN-SCENARIOS-1.md](./04A-CLUSTER-ADMIN-SCENARIOS-1.md) for RBAC, kubeadm, etcd

---

## Table of Contents
1. [Helm Scenarios (17-22)](#helm-scenarios)
2. [Kustomize Scenarios (23-26)](#kustomize-scenarios)
3. [CRDs and Operators (27-28)](#crds-and-operators)
4. [High Availability & Extensions (29-30)](#high-availability--extensions)

---

## Helm Scenarios

### Scenario 17: Install Helm

**Objective:** Install Helm CLI and verify.

**Your Task:**
```bash
# Install Helm and verify installation
```

<details>
<summary>💡 Solution</summary>

```bash
# Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Or via package manager
# Ubuntu/Debian
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# Verify installation
helm version

# Add common repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# List repos
helm repo list
```

</details>

---

### Scenario 18: Install a Helm Chart

**Objective:** Install an application using Helm.

**Requirements:**
- Install nginx from bitnami repo
- Release name: `my-nginx`
- Namespace: `web`

**Your Task:**
```bash
# Install nginx using Helm
```

<details>
<summary>💡 Solution</summary>

```bash
# Create namespace
kubectl create namespace web

# Search for chart
helm search repo nginx

# Show chart info
helm show chart bitnami/nginx
helm show values bitnami/nginx

# Install chart
helm install my-nginx bitnami/nginx -n web

# Or with custom values
helm install my-nginx bitnami/nginx -n web \
  --set replicaCount=2 \
  --set service.type=NodePort

# Verify
helm list -n web
kubectl get all -n web

# Get release status
helm status my-nginx -n web
```

</details>

---

### Scenario 19: Upgrade a Helm Release

**Objective:** Upgrade an existing Helm release.

**Your Task:**
```bash
# Upgrade the nginx release with new values
```

<details>
<summary>💡 Solution</summary>

```bash
# Check current values
helm get values my-nginx -n web

# Upgrade with new values
helm upgrade my-nginx bitnami/nginx -n web \
  --set replicaCount=3 \
  --set service.type=LoadBalancer

# Or using values file
cat <<EOF > nginx-values.yaml
replicaCount: 3
service:
  type: LoadBalancer
  port: 80
EOF

helm upgrade my-nginx bitnami/nginx -n web -f nginx-values.yaml

# Check history
helm history my-nginx -n web

# Verify changes
kubectl get pods -n web
kubectl get svc -n web
```

</details>

---

### Scenario 20: Rollback a Helm Release

**Objective:** Rollback to a previous Helm release version.

**Your Task:**
```bash
# Rollback to previous version
```

<details>
<summary>💡 Solution</summary>

```bash
# View history
helm history my-nginx -n web

# Output:
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete

# Rollback to revision 1
helm rollback my-nginx 1 -n web

# Verify
helm history my-nginx -n web
helm status my-nginx -n web

# Check rollback
kubectl get pods -n web
```

</details>

---

### Scenario 21: Uninstall a Helm Release

**Objective:** Remove a Helm release.

**Your Task:**
```bash
# Uninstall the nginx release
```

<details>
<summary>💡 Solution</summary>

```bash
# Uninstall release
helm uninstall my-nginx -n web

# Keep history for potential rollback
helm uninstall my-nginx -n web --keep-history

# Verify
helm list -n web
kubectl get all -n web

# If kept history, can still see it
helm list -n web -a
# Shows uninstalled releases
```

</details>

---

### Scenario 22: Create Custom Values File

**Objective:** Install chart with custom configuration.

**Your Task:**
```bash
# Install metrics-server with custom values
```

<details>
<summary>💡 Solution</summary>

```bash
# Get default values
helm show values metrics-server/metrics-server > metrics-values.yaml

# Or create custom values file
cat <<EOF > custom-metrics.yaml
args:
  - --kubelet-insecure-tls
  - --kubelet-preferred-address-types=InternalIP
replicas: 1
resources:
  requests:
    cpu: 100m
    memory: 200Mi
  limits:
    cpu: 100m
    memory: 200Mi
EOF

# Add repo and install
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server \
  -n kube-system \
  -f custom-metrics.yaml

# Verify
kubectl top nodes
```

</details>

---

## Kustomize Scenarios

### Scenario 23: Create Basic Kustomization

**Objective:** Use Kustomize to manage configurations.

**Your Task:**
```bash
# Create base Kustomize configuration
```

<details>
<summary>💡 Solution</summary>

```bash
# Create directory structure
mkdir -p kustomize/base
cd kustomize/base

# Create deployment
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF

# Create service
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
EOF

# Create kustomization.yaml
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
commonLabels:
  environment: base
EOF

# Preview
kubectl kustomize .

# Apply
kubectl apply -k .
```

</details>

---

### Scenario 24: Create Environment Overlays

**Objective:** Create dev and prod overlays.

**Your Task:**
```bash
# Create overlays for different environments
```

<details>
<summary>💡 Solution</summary>

```bash
# Create overlay directories
mkdir -p kustomize/overlays/dev
mkdir -p kustomize/overlays/prod

# Dev overlay
cat <<EOF > kustomize/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
namePrefix: dev-
resources:
- ../../base
replicas:
- name: webapp
  count: 1
commonLabels:
  environment: development
EOF

# Prod overlay
cat <<EOF > kustomize/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
namePrefix: prod-
resources:
- ../../base
replicas:
- name: webapp
  count: 3
commonLabels:
  environment: production
EOF

# Preview overlays
kubectl kustomize kustomize/overlays/dev
kubectl kustomize kustomize/overlays/prod

# Apply
kubectl create namespace dev
kubectl create namespace prod
kubectl apply -k kustomize/overlays/dev
kubectl apply -k kustomize/overlays/prod

# Verify
kubectl get all -n dev
kubectl get all -n prod
```

</details>

---

### Scenario 25: Use Kustomize Patches

**Objective:** Apply patches to modify resources.

**Your Task:**
```bash
# Create patches for resource modifications
```

<details>
<summary>💡 Solution</summary>

```bash
# Create a patch file
cat <<EOF > kustomize/overlays/prod/increase-memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  template:
    spec:
      containers:
      - name: webapp
        resources:
          limits:
            memory: 256Mi
          requests:
            memory: 128Mi
EOF

# Update kustomization to include patch
cat <<EOF > kustomize/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
namePrefix: prod-
resources:
- ../../base
replicas:
- name: webapp
  count: 3
commonLabels:
  environment: production
patches:
- path: increase-memory.yaml
EOF

# Preview and apply
kubectl kustomize kustomize/overlays/prod
kubectl apply -k kustomize/overlays/prod
```

</details>

---

### Scenario 26: Use ConfigMap Generator

**Objective:** Generate ConfigMaps with Kustomize.

**Your Task:**
```bash
# Generate ConfigMap using Kustomize
```

<details>
<summary>💡 Solution</summary>

```bash
# Create config file
cat <<EOF > kustomize/base/app.properties
database_url=localhost:5432
log_level=info
EOF

# Update base kustomization
cat <<EOF > kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
configMapGenerator:
- name: app-config
  files:
  - app.properties
- name: env-config
  literals:
  - LOG_LEVEL=debug
  - APP_NAME=webapp
EOF

# Preview generated ConfigMaps
kubectl kustomize kustomize/base

# Apply
kubectl apply -k kustomize/base

# Verify ConfigMaps (names have hash suffix)
kubectl get configmaps
```

</details>

---

## CRDs and Operators

### Scenario 27: Create a Custom Resource Definition

**Objective:** Create and use a simple CRD.

**Your Task:**
```bash
# Create a CRD and custom resource
```

<details>
<summary>💡 Solution</summary>

```yaml
# backup-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.example.com
spec:
  group: stable.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              schedule:
                type: string
              target:
                type: string
              retention:
                type: integer
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
    - bk
```

```bash
# Apply CRD
kubectl apply -f backup-crd.yaml

# Verify CRD created
kubectl get crd | grep backup

# Create custom resource
cat <<EOF | kubectl apply -f -
apiVersion: stable.example.com/v1
kind: Backup
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  target: /data
  retention: 7
EOF

# List backups
kubectl get backups
kubectl get bk  # Short name

# Describe
kubectl describe backup daily-backup
```

</details>

---

### Scenario 28: Install an Operator

**Objective:** Install and use a Kubernetes operator.

**Your Task:**
```bash
# Install cert-manager operator
```

<details>
<summary>💡 Solution</summary>

```bash
# Install cert-manager using kubectl
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Or using Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Verify operator pods
kubectl get pods -n cert-manager

# Check CRDs installed
kubectl get crd | grep cert-manager
# certificates.cert-manager.io
# clusterissuers.cert-manager.io
# issuers.cert-manager.io

# Create a self-signed issuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

# Verify
kubectl get clusterissuers
```

</details>

---

## High Availability & Extensions

### Scenario 29: Understand HA Control Plane

**Objective:** Understand HA topology options.

**Your Task:**
```bash
# Explore HA configurations
```

<details>
<summary>💡 Solution</summary>

```bash
# Check if cluster is HA
kubectl get nodes | grep control-plane

# View etcd topology (stacked vs external)
kubectl get pods -n kube-system | grep etcd
# If etcd pods exist on control plane = stacked topology

# Check kubeadm config for HA
kubectl get cm -n kube-system kubeadm-config -o yaml | grep -A5 controlPlane

# For external etcd, etcd endpoints are in API server config
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd

# View load balancer config (if present)
kubectl cluster-info
# Shows API server endpoint (load balancer in HA)
```

**HA Topologies:**

1. **Stacked etcd**: etcd runs on control plane nodes
   - Simpler setup
   - Less infrastructure
   - etcd failure = control plane failure

2. **External etcd**: Separate etcd cluster
   - More complex
   - Better isolation
   - Requires more nodes

</details>

---

### Scenario 30: Understand Extension Interfaces

**Objective:** Identify CNI, CSI, CRI in the cluster.

**Your Task:**
```bash
# Explore cluster extensions
```

<details>
<summary>💡 Solution</summary>

```bash
# CNI (Container Network Interface)
# Check which CNI is installed
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf

# Check CNI pods
kubectl get pods -A | grep -E "flannel|calico|weave|cilium"

# CSI (Container Storage Interface)
# List CSI drivers
kubectl get csidrivers

# Check storage classes using CSI
kubectl get sc -o yaml | grep provisioner

# CRI (Container Runtime Interface)
# Check container runtime
kubectl get nodes -o wide
# CONTAINER-RUNTIME column shows runtime

# On node, check runtime
sudo crictl info | grep -i runtime

# Check kubelet config for CRI
cat /var/lib/kubelet/config.yaml | grep containerRuntime
```

**Key Points:**
- **CNI**: Network plugin (Flannel, Calico, Cilium)
- **CSI**: Storage drivers (AWS EBS, GCE PD, etc.)
- **CRI**: Container runtime (containerd, CRI-O)

</details>

---

## Quick Reference - Helm

```bash
# Repository commands
helm repo add NAME URL
helm repo update
helm repo list
helm search repo KEYWORD

# Chart info
helm show chart CHART
helm show values CHART

# Install/Upgrade
helm install RELEASE CHART [-n NAMESPACE] [--set key=val] [-f values.yaml]
helm upgrade RELEASE CHART [-n NAMESPACE]
helm upgrade --install RELEASE CHART  # Install or upgrade

# Management
helm list [-n NAMESPACE] [-a]
helm status RELEASE [-n NAMESPACE]
helm history RELEASE [-n NAMESPACE]
helm rollback RELEASE REVISION [-n NAMESPACE]
helm uninstall RELEASE [-n NAMESPACE]
helm get values RELEASE [-n NAMESPACE]
```

## Quick Reference - Kustomize

```bash
# Preview output
kubectl kustomize DIR

# Apply directly
kubectl apply -k DIR

# kustomization.yaml structure
resources:      # List of resource files
namePrefix:     # Prefix for all resource names
namespace:      # Target namespace
commonLabels:   # Labels for all resources
replicas:       # Override replica count
patches:        # Strategic merge patches
configMapGenerator:  # Generate ConfigMaps
secretGenerator:     # Generate Secrets
```

---

**Total Cluster Admin Scenarios:** 30 (16 + 14)

**Next:** [05-NETWORKING-SCENARIOS.md](./05-NETWORKING-SCENARIOS.md) for Services & Networking
