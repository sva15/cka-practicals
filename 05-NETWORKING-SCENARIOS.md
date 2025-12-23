# CKA Services & Networking Scenarios (20% of Exam)

> **Exam Weight:** 20%  
> **Study Time:** ~22 hours across Days 3-5  
> **Total Scenarios:** 25

---

## Table of Contents
1. [Pod Connectivity (Scenarios 1-4)](#pod-connectivity)
2. [Services (Scenarios 5-12)](#services)
3. [Network Policies (Scenarios 13-18)](#network-policies)
4. [Ingress & Gateway API (Scenarios 19-22)](#ingress--gateway-api)
5. [CoreDNS (Scenarios 23-25)](#coredns)

---

## Pod Connectivity

### Scenario 1: Understand Pod Networking

**Objective:** Explore how pods communicate.

**Your Task:**
```bash
# Create pods and verify connectivity
```

<details>
<summary>💡 Solution</summary>

```bash
# Create two pods
kubectl run pod1 --image=nginx
kubectl run pod2 --image=busybox --restart=Never -- sleep 3600

# Get pod IPs
kubectl get pods -o wide
# pod1   10.244.1.5
# pod2   10.244.2.3

# Test connectivity from pod2 to pod1
kubectl exec pod2 -- wget -qO- --timeout=5 10.244.1.5

# Pods can communicate across nodes without NAT
# This is the Kubernetes networking model

# Check pod CIDR
kubectl cluster-info dump | grep -i podcidr

# Check CNI configuration
kubectl get pods -n kube-system | grep -E "flannel|calico|weave"
```

</details>

---

### Scenario 2: Pod DNS Resolution

**Objective:** Understand pod DNS naming.

**Your Task:**
```bash
# Test DNS resolution between pods
```

<details>
<summary>💡 Solution</summary>

```bash
# Create a service for pod1
kubectl expose pod pod1 --port=80 --name=pod1-svc

# From pod2, resolve by service name
kubectl exec pod2 -- nslookup pod1-svc
# Server: 10.96.0.10
# Address: 10.96.0.10:53
# Name: pod1-svc.default.svc.cluster.local

# Full DNS format
# <service>.<namespace>.svc.cluster.local

# Access via DNS
kubectl exec pod2 -- wget -qO- pod1-svc.default.svc.cluster.local

# Short form works within same namespace
kubectl exec pod2 -- wget -qO- pod1-svc
```

</details>

---

### Scenario 3: Cross-Namespace Communication

**Objective:** Connect pods across namespaces.

**Your Task:**
```bash
# Access service in different namespace
```

<details>
<summary>💡 Solution</summary>

```bash
# Create namespace and deployment
kubectl create namespace backend
kubectl create deployment db --image=nginx -n backend
kubectl expose deployment db --port=80 -n backend

# Create client in default namespace
kubectl run client --image=busybox --restart=Never -- sleep 3600

# Access service in another namespace
kubectl exec client -- wget -qO- --timeout=5 db.backend.svc.cluster.local

# Or shorter form
kubectl exec client -- wget -qO- --timeout=5 db.backend

# Verify DNS
kubectl exec client -- nslookup db.backend
```

</details>

---

### Scenario 4: Pod-to-External Communication

**Objective:** Allow pods to access external services.

**Your Task:**
```bash
# Test external connectivity from pods
```

<details>
<summary>💡 Solution</summary>

```bash
# Test external connectivity
kubectl exec client -- wget -qO- --timeout=10 http://httpbin.org/ip

# Check external DNS resolution
kubectl exec client -- nslookup google.com

# If external access fails:
# 1. Check NAT rules on nodes
# 2. Check if egress Network Policy blocks
# 3. Check CoreDNS can resolve external names

# Create ExternalName service for external access
kubectl create service externalname external-api \
  --external-name=api.example.com

# Access via service name
kubectl exec client -- nslookup external-api
# Returns api.example.com
```

</details>

---

## Services

### Scenario 5: Create ClusterIP Service

**Objective:** Create internal service.

**Requirements:**
- Deployment: `webapp` with nginx
- Service: ClusterIP on port 80

**Your Task:**
```bash
# Create ClusterIP service
```

<details>
<summary>💡 Solution</summary>

```bash
# Create deployment
kubectl create deployment webapp --image=nginx --replicas=3

# Imperative service creation
kubectl expose deployment webapp --port=80 --type=ClusterIP

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# Verify
kubectl get svc webapp
kubectl describe svc webapp

# Test
kubectl run tmp --rm -it --image=busybox -- wget -qO- webapp
```

</details>

---

### Scenario 6: Create NodePort Service

**Objective:** Expose service outside cluster.

**Requirements:**
- Expose webapp on a NodePort
- Specific NodePort: 30080

**Your Task:**
```bash
# Create NodePort service
```

<details>
<summary>💡 Solution</summary>

```bash
# Imperative (random port)
kubectl expose deployment webapp --port=80 --type=NodePort --name=webapp-np

# Declarative with specific port
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
EOF

# Verify
kubectl get svc webapp-nodeport
# PORT(S): 80:30080/TCP

# Access from any node IP
curl http://<NODE_IP>:30080

# Get node IPs
kubectl get nodes -o wide
```

**NodePort range:** 30000-32767

</details>

---

### Scenario 7: Create LoadBalancer Service

**Objective:** Create LoadBalancer type service.

**Your Task:**
```bash
# Create LoadBalancer service
```

<details>
<summary>💡 Solution</summary>

```bash
# Note: LoadBalancer requires cloud provider or MetalLB

# Imperative
kubectl expose deployment webapp --port=80 --type=LoadBalancer --name=webapp-lb

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-lb
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# Verify
kubectl get svc webapp-lb
# EXTERNAL-IP: <pending> without cloud provider
# EXTERNAL-IP: 1.2.3.4 with cloud provider

# For local testing, install MetalLB or use
# EXTERNAL-IP will remain <pending> on bare metal without LB
```

</details>

---

### Scenario 8: Create Headless Service

**Objective:** Create service without ClusterIP.

**Use Case:** StatefulSets, direct pod access

**Your Task:**
```bash
# Create headless service
```

<details>
<summary>💡 Solution</summary>

```bash
# Create headless service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-headless
spec:
  clusterIP: None  # This makes it headless
  selector:
    app: webapp
  ports:
  - port: 80
EOF

# Verify - no ClusterIP
kubectl get svc webapp-headless
# CLUSTER-IP: None

# DNS returns pod IPs directly
kubectl run tmp --rm -it --image=busybox -- nslookup webapp-headless
# Returns multiple A records for each pod IP
```

**Headless vs ClusterIP:**
- ClusterIP: Returns single virtual IP
- Headless: Returns pod IPs directly

</details>

---

### Scenario 9: Service with Named Ports

**Objective:** Use named ports in services.

**Your Task:**
```bash
# Create service with named target port
```

<details>
<summary>💡 Solution</summary>

```yaml
# named-port-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-named
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-named
  template:
    metadata:
      labels:
        app: web-named
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - name: http  # Named port
          containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-named-svc
spec:
  selector:
    app: web-named
  ports:
  - port: 80
    targetPort: http  # Reference by name
```

```bash
kubectl apply -f named-port-deploy.yaml
kubectl describe svc web-named-svc
# TargetPort: http/TCP
```

</details>

---

### Scenario 10: Multi-Port Service

**Objective:** Expose multiple ports on a service.

**Your Task:**
```bash
# Create service with multiple ports
```

<details>
<summary>💡 Solution</summary>

```yaml
# multi-port-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-svc
spec:
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  - name: metrics
    port: 9090
    targetPort: 9090
```

```bash
kubectl apply -f multi-port-svc.yaml
kubectl get svc multi-port-svc
# PORT(S): 80/TCP,443/TCP,9090/TCP
```

**Note:** When multiple ports, each port MUST have a name.

</details>

---

### Scenario 11: Debug Service Not Working

**Objective:** Troubleshoot service connectivity.

**Setup:** Create broken service
```bash
kubectl create deployment debug-app --image=nginx
kubectl create service clusterip debug-svc --tcp=80:80
```

**Your Task:**
```bash
# Service returns no response - debug
```

<details>
<summary>💡 Solution</summary>

```bash
# Step 1: Check service
kubectl get svc debug-svc
kubectl describe svc debug-svc

# Step 2: Check endpoints
kubectl get endpoints debug-svc
# ENDPOINTS: <none> - This is the problem!

# Step 3: Compare selectors
kubectl get svc debug-svc -o jsonpath='{.spec.selector}'
# {"app":"debug-svc"}

kubectl get deployment debug-app -o jsonpath='{.spec.selector.matchLabels}'
# {"app":"debug-app"}

# Labels don't match!

# Step 4: Fix selector
kubectl patch svc debug-svc -p '{"spec":{"selector":{"app":"debug-app"}}}'

# Or recreate service correctly
kubectl delete svc debug-svc
kubectl expose deployment debug-app --port=80 --name=debug-svc

# Verify
kubectl get endpoints debug-svc
# Now shows pod IPs
```

</details>

---

### Scenario 12: Session Affinity

**Objective:** Configure sticky sessions.

**Your Task:**
```bash
# Enable session affinity
```

<details>
<summary>💡 Solution</summary>

```bash
# Create service with session affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: sticky-svc
spec:
  selector:
    app: webapp
  ports:
  - port: 80
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
EOF

# Verify
kubectl describe svc sticky-svc | grep -i session
# Session Affinity: ClientIP

# Now requests from same client IP go to same pod
```

</details>

---

## Network Policies

### Scenario 13: Create Default Deny Policy

**Objective:** Block all traffic to pods.

**Your Task:**
```bash
# Create default deny ingress policy
```

<details>
<summary>💡 Solution</summary>

```yaml
# default-deny-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

```bash
kubectl apply -f default-deny-ingress.yaml

# Test - traffic should be blocked
kubectl run test --image=nginx
kubectl run client --image=busybox --restart=Never -- sleep 3600
kubectl exec client -- wget -qO- --timeout=3 test
# Timeout - blocked!
```

</details>

---

### Scenario 14: Allow Specific Pod Traffic

**Objective:** Allow traffic from specific pods.

**Requirements:**
- Allow traffic to `db` pod only from `web` pods

**Your Task:**
```bash
# Create policy to allow web->db traffic
```

<details>
<summary>💡 Solution</summary>

```bash
# Create pods
kubectl run db --image=nginx --labels=role=db
kubectl run web --image=busybox --labels=role=web --restart=Never -- sleep 3600
kubectl run other --image=busybox --labels=role=other --restart=Never -- sleep 3600

# Create policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: web
    ports:
    - port: 80
EOF

# Test
kubectl exec web -- wget -qO- --timeout=3 $(kubectl get pod db -o jsonpath='{.status.podIP}')
# Works!

kubectl exec other -- wget -qO- --timeout=3 $(kubectl get pod db -o jsonpath='{.status.podIP}')
# Timeout - blocked!
```

</details>

---

### Scenario 15: Allow From Namespace

**Objective:** Allow traffic from specific namespace.

**Your Task:**
```bash
# Allow traffic from monitoring namespace
```

<details>
<summary>💡 Solution</summary>

```bash
# Create namespaces
kubectl create namespace production
kubectl create namespace monitoring
kubectl label namespace monitoring purpose=monitoring

# Create app in production
kubectl run app --image=nginx -n production --labels=app=myapp

# Create policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
EOF

# Create test pods
kubectl run monitor --image=busybox -n monitoring --restart=Never -- sleep 3600
kubectl run other --image=busybox --restart=Never -- sleep 3600

# Test
kubectl exec -n monitoring monitor -- wget -qO- --timeout=3 \
  $(kubectl get pod app -n production -o jsonpath='{.status.podIP}')
# Works!

kubectl exec other -- wget -qO- --timeout=3 \
  $(kubectl get pod app -n production -o jsonpath='{.status.podIP}')
# Blocked!
```

</details>

---

### Scenario 16: Egress Network Policy

**Objective:** Control outbound traffic from pods.

**Your Task:**
```bash
# Allow egress only to specific destinations
```

<details>
<summary>💡 Solution</summary>

```yaml
# egress-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-to-db-only
spec:
  podSelector:
    matchLabels:
      role: web
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP
  # Allow to db pods
  - to:
    - podSelector:
        matchLabels:
          role: db
    ports:
    - port: 5432
```

```bash
kubectl apply -f egress-policy.yaml

# Web pods can only reach db on 5432 and DNS
# All other egress is blocked
```

</details>

---

### Scenario 17: Combined Ingress and Egress

**Objective:** Create policy with both directions.

**Your Task:**
```bash
# Create comprehensive network policy
```

<details>
<summary>💡 Solution</summary>

```yaml
# complete-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complete-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from web tier
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - port: 8080
  egress:
  # Allow to database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
```

```bash
kubectl apply -f complete-policy.yaml
```

</details>

---

### Scenario 18: Debug Network Policy

**Objective:** Troubleshoot blocked traffic.

**Your Task:**
```bash
# Debug why pods can't communicate
```

<details>
<summary>💡 Solution</summary>

```bash
# Step 1: List network policies
kubectl get networkpolicy -A

# Step 2: Describe specific policy
kubectl describe networkpolicy <policy-name>

# Step 3: Check pod labels
kubectl get pods --show-labels

# Step 4: Check namespace labels
kubectl get ns --show-labels

# Step 5: Verify CNI supports NetworkPolicy
# Flannel alone does NOT support NetworkPolicy
# Need Calico, Cilium, or Weave

# Common issues:
# 1. Selector doesn't match pod labels
# 2. Namespace selector missing namespace labels
# 3. Port mismatch
# 4. CNI doesn't support NetworkPolicy

# Test with temporary policy deletion
kubectl delete networkpolicy <policy-name>
# If traffic works now, policy was blocking
```

</details>

---

## Ingress & Gateway API

### Scenario 19: Install Ingress Controller

**Objective:** Deploy nginx ingress controller.

**Your Task:**
```bash
# Install nginx ingress controller
```

<details>
<summary>💡 Solution</summary>

```bash
# Using manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Or using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Wait for controller
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

</details>

---

### Scenario 20: Create Ingress Resource

**Objective:** Route traffic using Ingress.

**Requirements:**
- Route `/app1` to app1-svc
- Route `/app2` to app2-svc

**Your Task:**
```bash
# Create Ingress with path-based routing
```

<details>
<summary>💡 Solution</summary>

```bash
# Create backend deployments and services
kubectl create deployment app1 --image=nginx
kubectl expose deployment app1 --port=80

kubectl create deployment app2 --image=httpd
kubectl expose deployment app2 --port=80

# Create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
EOF

# Verify
kubectl get ingress
kubectl describe ingress multi-path-ingress

# Test
curl http://<INGRESS_IP>/app1
curl http://<INGRESS_IP>/app2
```

</details>

---

### Scenario 21: Ingress with TLS

**Objective:** Configure HTTPS with Ingress.

**Your Task:**
```bash
# Create Ingress with TLS termination
```

<details>
<summary>💡 Solution</summary>

```bash
# Create self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com"

# Create TLS secret
kubectl create secret tls myapp-tls --cert=tls.crt --key=tls.key

# Create Ingress with TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
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

# Test
curl -k https://myapp.example.com --resolve myapp.example.com:443:<INGRESS_IP>
```

</details>

---

### Scenario 22: Gateway API Basics

**Objective:** Use Gateway API for traffic routing.

**Your Task:**
```bash
# Create Gateway and HTTPRoute
```

<details>
<summary>💡 Solution</summary>

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Verify CRDs
kubectl get crd | grep gateway

# Create GatewayClass (provided by controller)
# For nginx:
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.io/ingress-nginx
EOF

# Create Gateway
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
EOF

# Create HTTPRoute
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app1
      port: 80
EOF

# Verify
kubectl get gateway,httproute
```

</details>

---

## CoreDNS

### Scenario 23: Explore CoreDNS Configuration

**Objective:** Understand CoreDNS setup.

**Your Task:**
```bash
# Examine CoreDNS configuration
```

<details>
<summary>💡 Solution</summary>

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS service
kubectl get svc -n kube-system kube-dns
# ClusterIP: 10.96.0.10 (default)

# View CoreDNS ConfigMap
kubectl get cm coredns -n kube-system -o yaml

# Key Corefile sections:
# .:53 { ... }           - Main server block
# kubernetes cluster.local - Handles cluster DNS
# forward . /etc/resolv.conf - External DNS forwarding
# cache 30               - DNS caching

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

</details>

---

### Scenario 24: Customize CoreDNS

**Objective:** Add custom DNS entries.

**Your Task:**
```bash
# Add custom DNS rewrite rule
```

<details>
<summary>💡 Solution</summary>

```bash
# Edit CoreDNS ConfigMap
kubectl edit cm coredns -n kube-system

# Add custom entries in Corefile:
# Option 1: Hosts plugin for static entries
# Add before "forward" line:
#     hosts {
#         10.0.0.100 custom.example.com
#         fallthrough
#     }

# Option 2: Rewrite plugin
#     rewrite name exact oldname.default.svc.cluster.local newname.default.svc.cluster.local

# After editing, restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# Test
kubectl run tmp --rm -it --image=busybox -- nslookup custom.example.com
```

</details>

---

### Scenario 25: Troubleshoot DNS Issues

**Objective:** Debug DNS resolution failures.

**Your Task:**
```bash
# Pods cannot resolve service names
```

<details>
<summary>💡 Solution</summary>

```bash
# Step 1: Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
# All should be Running

# Step 2: Check CoreDNS service
kubectl get svc kube-dns -n kube-system
kubectl get endpoints kube-dns -n kube-system
# Should have endpoints

# Step 3: Test DNS from debug pod
kubectl run dns-test --rm -it --image=busybox -- sh
# Inside pod:
nslookup kubernetes.default
cat /etc/resolv.conf
# nameserver should be 10.96.0.10

# Step 4: Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Step 5: Test direct CoreDNS query
kubectl exec dns-test -- nslookup kubernetes.default 10.96.0.10

# Common fixes:
# 1. Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# 2. Check ConfigMap for errors
kubectl get cm coredns -n kube-system -o yaml

# 3. Verify kube-dns service is correct
kubectl get svc kube-dns -n kube-system -o yaml
```

</details>

---

## Quick Reference

### Service Commands
```bash
# Create services
kubectl expose deployment NAME --port=P --type=TYPE
kubectl create service clusterip NAME --tcp=PORT:TARGET
kubectl create service nodeport NAME --tcp=PORT:TARGET --node-port=NP
kubectl create service loadbalancer NAME --tcp=PORT:TARGET

# Debug
kubectl get endpoints svcname
kubectl describe svc svcname
```

### Network Policy Tips
```bash
# Check if CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep calico
# Flannel alone does NOT support NetworkPolicy

# Empty podSelector = all pods
spec:
  podSelector: {}

# Empty ingress/egress = deny all
spec:
  policyTypes: [Ingress]
  # No ingress: field = deny all ingress
```

### DNS Format
```
<service>.<namespace>.svc.cluster.local
<pod-ip-dashed>.<namespace>.pod.cluster.local
```

---

**Total Networking Scenarios:** 25

**Scenario Totals Across All Files:**
- Storage: 18
- Troubleshooting: 38
- Workloads & Scheduling: 22
- Cluster Admin (Part 1 + 2): 30
- Services & Networking: 25
- **Grand Total: 133 scenarios**

---

**Next:** [06-KUBECTL-CHEATSHEET.md](./06-KUBECTL-CHEATSHEET.md)
