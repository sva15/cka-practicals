# CKA Exam Tips & Strategies 📋

> **Maximize Your Score with These Strategies**  
> Study Time: Review this the day before the exam

---

## Exam Format Overview

| Aspect | Details |
|--------|---------|
| Duration | 2 hours |
| Questions | 15-20 performance-based tasks |
| Passing Score | 66% |
| Environment | Remote proctored |
| Open Book | kubernetes.io/docs allowed |
| Retake | One free retake included |
| Kubernetes Version | Check CNCF site for current version |

---

## Before the Exam

### Environment Setup
- [ ] Stable internet connection (wired preferred)
- [ ] Quiet room with clear desk
- [ ] Government-issued ID ready
- [ ] Water in clear container allowed
- [ ] Close all other applications
- [ ] Test webcam and microphone

### Mental Preparation
- [ ] Good night's sleep (7-8 hours)
- [ ] Light meal before exam
- [ ] Bathroom break before starting
- [ ] Stay calm - you've prepared!

---

## First 5 Minutes: Terminal Setup

**Do this immediately when exam starts:**

```bash
# 1. Set up aliases (saves LOTS of time)
alias k=kubectl
alias kn='kubectl -n'

# 2. Enable autocompletion
source <(kubectl completion bash)
complete -F __start_kubectl k

# 3. Set up dry-run shortcut
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# 4. Verify setup works
k get nodes
```

---

## Time Management Strategy

### Question Approach

| Difficulty | Time | Action |
|------------|------|--------|
| Easy | 3-5 min | Do immediately |
| Medium | 5-8 min | Do if comfortable |
| Hard | 8-12 min | Flag and return |

### Time Allocation
- **First pass (60 min):** Answer all easy/medium questions
- **Second pass (40 min):** Tackle hard questions
- **Review (20 min):** Verify completed work

### Flag and Skip
- If stuck for more than 2 minutes, **FLAG and MOVE ON**
- Come back to difficult questions later
- Don't let one question consume too much time

---

## Context Switching (CRITICAL!)

⚠️ **Most common mistake: Wrong cluster/namespace**

### Before EVERY Question
```bash
# 1. Read the question carefully
# 2. Switch to correct context (if specified)
kubectl config use-context <context-name>

# 3. Set namespace (if specified)
kubectl config set-context --current --namespace=<ns>

# Or use -n flag with every command
kubectl get pods -n <namespace>
```

### Verify Your Context
```bash
kubectl config current-context
kubectl config view --minify | grep namespace
```

---

## Speed Tips

### Use Imperative Commands
```bash
# Faster than writing YAML:
k run nginx --image=nginx
k create deploy nginx --image=nginx --replicas=3
k expose deploy nginx --port=80
k create cm myconfig --from-literal=key=value
k create secret generic mysecret --from-literal=pass=secret
k create sa mysa
k create role myrole --verb=get --resource=pods
k create rolebinding bind --role=myrole --serviceaccount=default:mysa
```

### Generate YAML When Needed
```bash
# Generate and modify
k run nginx --image=nginx $do > pod.yaml
vim pod.yaml
k apply -f pod.yaml

# Or use kubectl create with --dry-run
k create deploy nginx --image=nginx $do > deploy.yaml
```

### Quick Pod for Testing
```bash
# Test connectivity
k run tmp --rm -it --image=busybox -- wget -qO- http://service-name

# Test DNS
k run tmp --rm -it --image=busybox -- nslookup kubernetes.default
```

---

## Using Documentation Effectively

### Allowed Resources
- kubernetes.io/docs
- kubernetes.io/blog (rarely needed)
- kubernetes.io/docs/reference (API reference)

### Efficient Searching
1. Use browser's **Ctrl+F** to search
2. Bookmark key pages before exam
3. Copy-paste YAML templates

### Key Pages to Bookmark
```
https://kubernetes.io/docs/reference/kubectl/quick-reference/
https://kubernetes.io/docs/concepts/workloads/pods/
https://kubernetes.io/docs/concepts/services-networking/service/
https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
https://kubernetes.io/docs/reference/access-authn-authz/rbac/
https://kubernetes.io/docs/concepts/services-networking/network-policies/
```

---

## Question Categories & Strategies

### 1. Pod/Deployment Creation (Easy)
- Use imperative commands
- Generate YAML only if complex

```bash
k run nginx --image=nginx --port=80 --labels=app=web
k create deploy webapp --image=nginx --replicas=3
```

### 2. Service Exposure (Easy)
- Check existing labels first
- Use `k expose` command

```bash
k get pods --show-labels
k expose pod nginx --port=80 --type=NodePort
```

### 3. RBAC (Medium)
- Create role first, then binding
- Use imperative commands

```bash
k create role pod-reader --verb=get,list --resource=pods
k create rolebinding dev-reader --role=pod-reader --serviceaccount=dev:dev-sa
```

### 4. etcd Backup/Restore (Medium-Hard)
- Know cert paths by heart
- Practice the full workflow

```bash
# Backup
etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 5. Cluster Upgrade (Medium)
- Control plane first, then workers
- Drain before upgrade

### 6. Troubleshooting (Variable)
- Follow systematic approach
- Check events, logs, status

```bash
k describe pod <name>      # Check events
k logs <pod> --previous    # Previous container
k get events               # Recent events
```

### 7. Network Policies (Medium)
- Check if CNI supports policies
- Test after applying

---

## Common Mistakes to Avoid

### 1. Wrong Context/Namespace
✅ Always verify context first
✅ Use `-n namespace` flag

### 2. Not Reading Question Completely
✅ Read ENTIRE question before starting
✅ Note specific requirements (names, namespaces, labels)

### 3. Typos in Resource Names
✅ Copy names from question when possible
✅ Use tab completion

### 4. Forgetting to Verify
✅ Always run `k get` after creating
✅ Check pod is Running, not Pending

### 5. Overcomplicating Answers
✅ Use imperative when possible
✅ Don't add unnecessary features

### 6. Spending Too Long on One Question
✅ Flag and move on after 5-7 minutes
✅ Return with fresh perspective

---

## Verification Checklist

After completing each question:

- [ ] Resource exists: `k get <resource>`
- [ ] Correct namespace: Check `-n` flag
- [ ] Pod is Running: Not Pending/Error
- [ ] Labels match selectors (for services)
- [ ] Test connectivity if applicable

---

## Quick Debug Commands

```bash
# Pod issues
k describe pod <name>
k logs <pod> --previous
k get events --sort-by='.lastTimestamp'

# Service issues
k get endpoints <svc>
k describe svc <svc>

# Node issues
k describe node <name>
journalctl -u kubelet -n 100

# Check permissions
k auth can-i <verb> <resource> --as=<user>
```

---

## Score Optimization

### Partial Credit
- Some questions have multiple parts
- Complete what you can
- Partial solutions still earn points

### Priority Order
1. ✅ Answer all easy questions first
2. ✅ Then medium difficulty
3. ✅ Hard questions last
4. ✅ Review and verify

### Don't Leave Blanks
- Attempt every question
- Even partial answers score points

---

## Day-of-Exam Checklist

### 1 Hour Before
- [ ] Quiet room ready
- [ ] Desk cleared
- [ ] ID accessible
- [ ] Water in clear container
- [ ] Bathroom break taken
- [ ] Computer restarted
- [ ] Close unnecessary apps

### 15 Minutes Before
- [ ] Launch exam platform
- [ ] Complete check-in process
- [ ] Test audio/video
- [ ] Read exam rules

### During Exam
- [ ] Set up aliases first
- [ ] Check context before each question
- [ ] Use imperative commands
- [ ] Flag hard questions
- [ ] Verify your work
- [ ] Don't panic!

---

## Emergency Recovery

### If You're Stuck
1. Take a deep breath
2. Re-read the question
3. Check kubernetes.io docs
4. Try a simpler approach
5. Flag and move on

### If Terminal Freezes
- Refresh the browser
- Contact proctor if needed
- Clock pauses for technical issues

### If Time Running Out
- Focus on easy questions first
- Partial answers are okay
- Submit what you have

---

## Final Words

- **You've prepared** - Trust your preparation
- **Stay calm** - Anxiety wastes time
- **Read carefully** - Most mistakes are from rushing
- **Verify everything** - A few seconds of verification saves points
- **Use documentation** - It's open book for a reason
- **Don't give up** - Keep working until time ends

---

**You've got this! Good luck! 🎯🚀**
