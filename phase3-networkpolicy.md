# Phase 3 — NetworkPolicy

> **Goal:** Build a zero-trust network architecture with label-based pod isolation. Understand how NetworkPolicy translates to kernel-level packet filtering.  
> **Time:** ~4-5 hours  
> **Output:** 3-tier app with verified east-west traffic restriction

---

## The Problem with Default Kubernetes Networking

By default, **every pod can reach every other pod** in the cluster — no restrictions.  
In a real application with frontend, backend, and database tiers, this means:
- A compromised frontend can directly attack your database
- Any pod can exfiltrate data to any other pod
- No segmentation between tenants or environments

**NetworkPolicy** fixes this. It's Kubernetes-native firewall rules at the pod level.

---

## How NetworkPolicy Works

```
Without policy:  frontend ↔ backend ↔ database ↔ everything
With policy:     frontend → backend → database (only these paths)
```

Key rules:
1. Policies are **additive** — you can only allow, never explicitly deny
2. A pod with **no matching policy** gets all traffic allowed
3. A pod with **any matching policy** gets only traffic that policy allows
4. Blocked traffic is **silently dropped** (no RST, no error)

---

## Pre-Check

```bash
k get pods
# pod-a and pod-b from Phase 2 should still be running
```

---

## Step 1: Document the Open State (Baseline)

```bash
POD_B_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod pod-b -o jsonpath='{.status.podIP}')
docker exec k8s-net-lab-control-plane kubectl exec pod-a -- ping -c 2 $POD_B_IP && echo "✅ OPEN — all traffic allowed"
```

---

## Step 2: Apply Default-Deny Policy

```bash
cat <<EOF > default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

docker cp default-deny.yaml k8s-net-lab-control-plane:/default-deny.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /default-deny.yaml
```

> `podSelector: {}` = select ALL pods in namespace.  
> No ingress/egress rules = nothing is allowed through.

### Verify traffic is now blocked
```bash
POD_B_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod pod-b -o jsonpath='{.status.podIP}')
docker exec k8s-net-lab-control-plane kubectl exec pod-a -- ping -c 2 $POD_B_IP && echo "reachable" || echo "❌ BLOCKED"
```

Expected: `❌ BLOCKED` — 100% packet loss, silently dropped.

---

## Step 3: Deploy 3-Tier Application

```bash
cat <<EOF > three-tier.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    tier: backend
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    tier: database
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

docker cp three-tier.yaml k8s-net-lab-control-plane:/three-tier.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /three-tier.yaml
```

Wait for all 3 pods and note their IPs:
```bash
k get pods -o wide -w
```

---

## Step 4: Apply Allow Policies

```bash
cat <<EOF > allow-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
EOF

docker cp allow-policies.yaml k8s-net-lab-control-plane:/allow-policies.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /allow-policies.yaml
```

Policy summary:

| Policy | Guards | Allows In From |
|--------|--------|----------------|
| `allow-frontend-to-backend` | backend | only `tier=frontend` |
| `allow-backend-to-database` | database | only `tier=backend` |
| `allow-backend-egress` | backend outbound | only to `tier=database` |
| `allow-frontend-egress` | frontend outbound | only to `tier=backend` |

---

## Step 5: Verify All Traffic Paths

```bash
FRONTEND_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod frontend -o jsonpath='{.status.podIP}')
BACKEND_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod backend -o jsonpath='{.status.podIP}')
DATABASE_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod database -o jsonpath='{.status.podIP}')
```

```bash
# ✅ Should WORK
docker exec k8s-net-lab-control-plane kubectl exec frontend -- ping -c 2 $BACKEND_IP \
  && echo "✅ frontend→backend ALLOWED" || echo "❌ BLOCKED"

# ✅ Should WORK
docker exec k8s-net-lab-control-plane kubectl exec backend -- ping -c 2 $DATABASE_IP \
  && echo "✅ backend→database ALLOWED" || echo "❌ BLOCKED"

# ❌ Should FAIL — skipping the backend tier
docker exec k8s-net-lab-control-plane kubectl exec frontend -- ping -c 2 $DATABASE_IP \
  && echo "✅ ALLOWED" || echo "❌ frontend→database BLOCKED"

# ❌ Should FAIL — wrong direction
docker exec k8s-net-lab-control-plane kubectl exec database -- ping -c 2 $BACKEND_IP \
  && echo "✅ ALLOWED" || echo "❌ database→backend BLOCKED"
```

Expected results:
```
✅ frontend→backend ALLOWED
✅ backend→database ALLOWED
❌ frontend→database BLOCKED
❌ database→backend BLOCKED
```

---

## Step 6: Audit Your Policies

```bash
# List all active policies
k get networkpolicies

# Describe a specific policy
k describe networkpolicy allow-frontend-to-backend
```

Expected describe output:
```
PodSelector:     tier=backend
Allowing ingress traffic:
  From: PodSelector: tier=frontend
  To Port: <any>
Not affecting egress traffic
Policy Types: Ingress
```

### View iptables rules Calico created
```bash
WORKER2_PID=$(docker inspect k8s-net-lab-worker2 --format '{{.State.Pid}}')
sudo nsenter -t $WORKER2_PID -n iptables -L --line-numbers | grep -A 3 "cali"
```

---

## Step 7: Cleanup Phase 3

```bash
docker exec k8s-net-lab-control-plane kubectl delete pods frontend backend database pod-a pod-b
docker exec k8s-net-lab-control-plane kubectl delete networkpolicies --all

# Verify clean
k get pods
k get networkpolicies
# Both: No resources found
```

---

## What You Learned

| Concept | Evidence |
|---------|----------|
| Default-deny baseline | Empty podSelector blocks all traffic |
| Label-based policy | Traffic rules tied to pod labels not IPs |
| Ingress vs Egress | Separate control of inbound and outbound |
| Policy is additive | Can only allow, never explicitly deny |
| Silent drops | Blocked traffic = no RST, just packet loss |
| Policy audit | kubectl describe shows what's actually enforced |

---

## Common Mistake: Label Mismatch

If a NetworkPolicy `podSelector` references a label that doesn't exist on any pod, the policy silently applies to nothing. Always verify:

```bash
k get pods --show-labels
```

If the label is missing:
```bash
docker exec k8s-net-lab-control-plane kubectl label pod <pod-name> <key>=<value>
```

See [issues.md](issues.md) — `Issue 7` for details.

---

➡️ **Next:** [Phase 4 — Services & DNS](phase4-services-dns.md)
