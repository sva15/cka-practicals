# CKA kubectl Cheatsheet 🚀

> **Quick Reference for CKA Exam**  
> Print this and memorize before the exam!

---

## Exam Setup (First Thing!)

```bash
# Set up aliases and autocompletion
alias k=kubectl
alias kn='kubectl -n'
alias kall='kubectl get all -A'

# Enable autocompletion
source <(kubectl completion bash)
complete -F __start_kubectl k

# Shortcut for dry-run output
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Example usage:
# k run nginx --image=nginx $do > pod.yaml
# k delete pod nginx $now
```

---

## Core Commands

### Get Resources
```bash
k get pods                      # List pods
k get pods -o wide              # More details (node, IP)
k get pods -A                   # All namespaces
k get pods -l app=nginx         # By label
k get pods --show-labels        # Show labels
k get pods -o yaml              # YAML output
k get pods -o json              # JSON output
k get all                       # All resource types
k get events --sort-by='.lastTimestamp'  # Recent events
```

### Describe & Inspect
```bash
k describe pod <name>           # Detailed info
k describe node <name>          # Node details
k logs <pod>                    # Container logs
k logs <pod> -c <container>     # Specific container
k logs <pod> --previous         # Previous container
k logs <pod> -f                 # Follow logs
```

### Create Resources
```bash
# Quick pod creation
k run nginx --image=nginx
k run nginx --image=nginx --port=80
k run nginx --image=nginx --labels=app=web,tier=frontend
k run nginx --image=nginx --env=VAR=value
k run nginx --image=nginx $do > pod.yaml

# Deployment
k create deployment nginx --image=nginx
k create deployment nginx --image=nginx --replicas=3

# Service
k expose pod nginx --port=80 --type=ClusterIP
k expose deployment nginx --port=80 --type=NodePort
k create service clusterip nginx --tcp=80:80

# ConfigMap/Secret
k create configmap myconfig --from-literal=key=value
k create configmap myconfig --from-file=config.txt
k create secret generic mysecret --from-literal=pass=secret

# Namespace
k create namespace dev

# ServiceAccount
k create serviceaccount mysa

# RBAC
k create role myrole --verb=get,list --resource=pods
k create rolebinding mybinding --role=myrole --serviceaccount=default:mysa
k create clusterrole myrole --verb=get,list --resource=nodes
k create clusterrolebinding mybinding --clusterrole=myrole --user=admin
```

### Modify Resources
```bash
k edit pod <name>               # Edit in vim
k apply -f file.yaml            # Apply changes
k replace -f file.yaml          # Replace resource
k patch pod <name> -p '{"spec":{"containers":[{"name":"nginx","image":"nginx:1.23"}]}}'

# Deployments
k scale deployment nginx --replicas=5
k set image deployment/nginx nginx=nginx:1.23
k rollout status deployment/nginx
k rollout history deployment/nginx
k rollout undo deployment/nginx
k rollout undo deployment/nginx --to-revision=2

# Labels
k label pod nginx app=web
k label pod nginx app-              # Remove label
k label node node1 disktype=ssd

# Annotations
k annotate pod nginx description="My app"

# Taints
k taint node node1 key=value:NoSchedule
k taint node node1 key=value:NoSchedule-  # Remove
```

### Delete Resources
```bash
k delete pod nginx
k delete pod nginx $now         # Force delete
k delete pods -l app=nginx      # By label
k delete pods --all             # All pods
k delete -f file.yaml           # From file
```

---

## Resource-Specific Commands

### Pods
```bash
k run nginx --image=nginx --restart=Never   # Pod only
k run nginx --image=nginx --command -- sleep 3600
k exec nginx -- ls /                        # Run command
k exec -it nginx -- /bin/sh                 # Interactive shell
k cp nginx:/path/file ./local               # Copy files
k port-forward pod/nginx 8080:80            # Port forward
```

### Deployments
```bash
k create deploy nginx --image=nginx --replicas=3
k scale deploy nginx --replicas=5
k autoscale deploy nginx --min=2 --max=10 --cpu-percent=80
k set resources deploy nginx --requests=cpu=100m,memory=128Mi
k set env deploy nginx ENV_VAR=value
k rollout pause deploy/nginx
k rollout resume deploy/nginx
```

### Jobs/CronJobs
```bash
k create job myjob --image=busybox -- date
k create cronjob mycron --image=busybox --schedule="*/5 * * * *" -- date
k create job manual --from=cronjob/mycron   # Manual trigger
```

### Services
```bash
k expose deploy nginx --port=80 --type=ClusterIP
k expose pod nginx --port=80 --type=NodePort
k expose deploy nginx --port=80 --type=LoadBalancer
k get endpoints nginx                        # Check endpoints
```

### ConfigMaps/Secrets
```bash
# Create
k create cm myconfig --from-literal=key=val
k create cm myconfig --from-file=config.txt
k create secret generic mysecret --from-literal=pass=secret
k create secret tls mytls --cert=cert.pem --key=key.pem

# View
k get cm myconfig -o yaml
k get secret mysecret -o jsonpath='{.data.pass}' | base64 -d
```

---

## Troubleshooting Commands

### Debug Pods
```bash
k describe pod <name>           # Check events
k logs <pod>                    # Check logs
k logs <pod> --previous         # Previous container
k get pod <name> -o yaml        # Full spec
k get events --field-selector involvedObject.name=<pod>
```

### Debug Nodes
```bash
k describe node <name>
k get node <name> -o yaml
k top node                      # Resource usage
k cordon <node>                 # Mark unschedulable
k uncordon <node>               # Mark schedulable
k drain <node> --ignore-daemonsets --delete-emptydir-data
```

### Debug Services
```bash
k get endpoints <svc>           # Check endpoints
k describe svc <svc>            # Check selector
k get svc -o wide               # See ports
```

### Check Permissions
```bash
k auth can-i create pods
k auth can-i create pods --as=jane
k auth can-i list nodes --as=system:serviceaccount:ns:sa
k auth can-i --list             # All permissions
```

---

## Output Formatting

### JSONPath
```bash
# Get pod IPs
k get pods -o jsonpath='{.items[*].status.podIP}'

# Get node names
k get nodes -o jsonpath='{.items[*].metadata.name}'

# Get container images
k get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Get pod by condition
k get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Custom output with ranges
k get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Custom Columns
```bash
k get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'
k get pods -o custom-columns='POD:.metadata.name,IMAGE:.spec.containers[0].image'
```

### Sorting
```bash
k get pods --sort-by=.metadata.name
k get pods --sort-by=.status.startTime
k get pv --sort-by=.spec.capacity.storage
```

---

## Quick YAML Generation

```bash
# Pod
k run nginx --image=nginx $do > pod.yaml

# Deployment
k create deploy nginx --image=nginx $do > deploy.yaml

# Service
k expose deploy nginx --port=80 $do > svc.yaml

# Job
k create job myjob --image=busybox $do -- date > job.yaml

# ConfigMap
k create cm myconfig --from-literal=key=value $do > cm.yaml

# RBAC
k create role myrole --verb=get --resource=pods $do > role.yaml
k create rolebinding bind --role=myrole --user=jane $do > rb.yaml
```

---

## Imperative vs Declarative

| Task | Imperative (Faster) | Declarative |
|------|---------------------|-------------|
| Create pod | `k run nginx --image=nginx` | `k apply -f pod.yaml` |
| Create deploy | `k create deploy nginx --image=nginx` | `k apply -f deploy.yaml` |
| Expose svc | `k expose deploy nginx --port=80` | `k apply -f svc.yaml` |
| Scale | `k scale deploy nginx --replicas=5` | `k apply -f deploy.yaml` |
| Update image | `k set image deploy/nginx nginx=nginx:1.23` | `k apply -f deploy.yaml` |

**Exam Tip:** Use imperative for speed, declarative for complex configs.

---

## Namespace Shortcuts

```bash
# Set default namespace
k config set-context --current --namespace=dev

# Or use -n flag
k get pods -n kube-system
k get pods -A                   # All namespaces

# Create resource in namespace
k run nginx --image=nginx -n dev
```

---

## Context & Cluster

```bash
k config view                    # View config
k config current-context         # Current context
k config get-contexts            # List contexts
k config use-context <name>      # Switch context
k config set-context --current --namespace=dev
```

---

## Documentation Lookup

```bash
# Explain resource fields
k explain pod
k explain pod.spec
k explain pod.spec.containers
k explain pod.spec.containers.resources

# API resources
k api-resources                  # All resources
k api-resources --namespaced=true
k api-resources | grep -i deploy

# API versions
k api-versions
```

---

## Key File Locations

| Component | Location |
|-----------|----------|
| kubeconfig | `~/.kube/config` |
| Static pods | `/etc/kubernetes/manifests/` |
| Kubelet config | `/var/lib/kubelet/config.yaml` |
| PKI certs | `/etc/kubernetes/pki/` |
| etcd certs | `/etc/kubernetes/pki/etcd/` |
| CNI config | `/etc/cni/net.d/` |
| CNI binaries | `/opt/cni/bin/` |

---

## etcd Commands

```bash
export ETCDCTL_API=3

# Backup
etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# Health check
etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Exam Quick Tips

1. **Always check namespace:** `k config set-context --current --namespace=<ns>`
2. **Generate YAML:** Use `$do` (--dry-run=client -o yaml)
3. **Force delete:** Use `$now` (--force --grace-period=0)
4. **Use documentation:** Ctrl+F to search kubernetes.io/docs
5. **Check events:** `k describe` shows events at bottom
6. **Verify work:** Always verify with `k get` after changes

---

**Good luck with your CKA exam! 🎯**
