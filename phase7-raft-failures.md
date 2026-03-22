# Phase 7 — Raft Cluster & Failure Modes

> **Goal:** Deploy a 3-node etcd Raft cluster and deliberately cause network failures to document failure modes, observable symptoms, and mitigations.  
> **Time:** ~5-8 hours  
> **Output:** 3 failure modes documented with logs, evidence, and mitigations

---

## What is Raft?

Raft is a distributed consensus algorithm used by etcd, CockroachDB, TiKV, and Kubernetes itself (etcd stores all cluster state).

In Raft:
- One node is the **leader** — it accepts all writes
- Other nodes are **followers** — they replicate the leader's log
- A write is committed only when a **quorum** (majority) of nodes acknowledge it
- If the leader disappears, followers hold an **election** to pick a new leader

For a 3-node cluster:
- Quorum = 2 nodes
- Can tolerate **1 node failure** and continue operating
- With 2 nodes down = no quorum = cluster stops accepting writes

---

## Step 1: Deploy etcd StatefulSet

```bash
cat <<EOF > etcd-cluster.yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd
  namespace: default
  labels:
    app: etcd
spec:
  clusterIP: None
  selector:
    app: etcd
  ports:
  - name: client
    port: 2379
  - name: peer
    port: 2380
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
  namespace: default
spec:
  serviceName: etcd
  replicas: 3
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.0
        command:
        - etcd
        - --name=\$(POD_NAME)
        - --initial-advertise-peer-urls=http://\$(POD_NAME).etcd:2380
        - --listen-peer-urls=http://0.0.0.0:2380
        - --listen-client-urls=http://0.0.0.0:2379
        - --advertise-client-urls=http://\$(POD_NAME).etcd:2379
        - --initial-cluster=etcd-0=http://etcd-0.etcd:2380,etcd-1=http://etcd-1.etcd:2380,etcd-2=http://etcd-2.etcd:2380
        - --initial-cluster-state=new
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
EOF

docker cp etcd-cluster.yaml k8s-net-lab-control-plane:/etcd-cluster.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /etcd-cluster.yaml
```

Watch sequential startup (StatefulSets start pods in order):
```bash
k get pods -w | grep etcd
# etcd-0 starts first, then etcd-1, then etcd-2
# This is intentional — Raft needs etcd-0 to bootstrap before others join
```

---

## Step 2: Verify Raft Leader Election

```bash
docker exec k8s-net-lab-control-plane kubectl exec etcd-0 -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --endpoints=http://etcd-0.etcd:2379,http://etcd-1.etcd:2379,http://etcd-2.etcd:2379 \
  endpoint status -w table"
```

Expected:
```
+-------------------------+------------------+---------+----------+-----------+------------+-----------+
|        ENDPOINT         |        ID        | VERSION | DB SIZE  | IS LEADER | RAFT TERM  | RAFT INDEX|
+-------------------------+------------------+---------+----------+-----------+------------+-----------+
| http://etcd-0.etcd:2379 | 69691ed70da97612 |  3.5.0  |  20 kB   |   true    |     2      |     8     |
| http://etcd-1.etcd:2379 | 7e44220b60d58d6a |  3.5.0  |  20 kB   |   false   |     2      |     8     |
| http://etcd-2.etcd:2379 | 63cdb3774b98cc2e |  3.5.0  |  20 kB   |   false   |     2      |     8     |
+-------------------------+------------------+---------+----------+-----------+------------+-----------+
```

All nodes at the same `RAFT INDEX` = fully synchronized.

---

## Step 3: Write and Replicate Test Data

```bash
# Write to etcd-0 (the leader)
docker exec k8s-net-lab-control-plane kubectl exec etcd-0 -- sh -c \
  "ETCDCTL_API=3 etcdctl --endpoints=http://etcd-0.etcd:2379 put /test/phase7 'raft-is-working'"
# Expected: OK

# Read from etcd-2 (a follower on a different node)
docker exec k8s-net-lab-control-plane kubectl exec etcd-2 -- sh -c \
  "ETCDCTL_API=3 etcdctl --endpoints=http://etcd-2.etcd:2379 get /test/phase7"
# Expected: raft-is-working
```

Write committed on leader → replicated to followers → readable from any node.

---

## Failure Mode 1: Leader Node Loss

### What We're Testing
Delete the leader pod (simulating a node crash or OOM kill) and observe Raft elect a new leader.

### Identify the Leader
```bash
docker exec k8s-net-lab-control-plane kubectl exec etcd-0 -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --endpoints=http://etcd-0.etcd:2379,http://etcd-1.etcd:2379,http://etcd-2.etcd:2379 \
  endpoint status -w table"
# Note which pod has IS LEADER: true
```

### Option A: NetworkPolicy Isolation (softer)
```bash
cat <<EOF > isolate-leader.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-etcd-leader
  namespace: default
spec:
  podSelector:
    matchLabels:
      statefulset.kubernetes.io/pod-name: etcd-0
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  ingress: []
EOF

docker cp isolate-leader.yaml k8s-net-lab-control-plane:/isolate-leader.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /isolate-leader.yaml
```

### Option B: Pod Deletion (harder crash, triggers faster)
```bash
docker exec k8s-net-lab-control-plane kubectl delete pod etcd-0
```

### Watch the Election (Terminal 2)
```bash
watch -n 1 'docker exec k8s-net-lab-control-plane kubectl exec etcd-1 -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --endpoints=http://etcd-1.etcd:2379,http://etcd-2.etcd:2379 \
  endpoint status -w json 2>/dev/null" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); \
  [print(x[\"Endpoint\"], \"leader:\", hex(x[\"Status\"][\"leader\"]), \
  \"term:\", x[\"Status\"][\"raftTerm\"]) for x in d]"'
```

### Election Log Evidence (Terminal 1)
```bash
docker exec k8s-net-lab-control-plane kubectl logs etcd-1 | \
  grep -E "became|elected|election|vote|term"
```

Expected log sequence:
```
received MsgTimeoutNow from etcd-0          ← old leader steps down
7e44220b60d58d6a is starting a new election at term 2
7e44220b60d58d6a became candidate at term 3  ← etcd-1 campaigns
sent MsgVote request to 63cdb3774b98cc2e     ← asks etcd-2 for vote
received MsgVoteResp (2 votes)               ← quorum achieved
7e44220b60d58d6a became leader at term 3     ← NEW LEADER
```

### Verify Zero Data Loss
```bash
docker exec k8s-net-lab-control-plane kubectl exec etcd-2 -- sh -c \
  "ETCDCTL_API=3 etcdctl --endpoints=http://etcd-2.etcd:2379 get /test/phase7"
# Expected: raft-is-working (data written before crash is preserved)
```

### Remove Isolation
```bash
docker exec k8s-net-lab-control-plane kubectl delete networkpolicy isolate-etcd-leader
```

---

## Failure Mode 1 Documentation

```
FAILURE MODE 1: Leader Node Loss
─────────────────────────────────
Trigger:     etcd leader pod deleted or network isolated
Observable:
  - etcd-0 logs: "prober detected unhealthy status" for peers
  - etcd-1 logs: "became candidate", "became leader"
  - RAFT TERM increments (2 → 3)
  - New IS LEADER in endpoint status table

Timeline:
  T+0ms    Leader deleted / isolated
  T+0ms    Leader sends MsgTimeoutNow (graceful stepdown if possible)
  T+4ms    Follower becomes candidate at term 3
  T+4ms    Second follower casts vote
  T+4ms    Quorum achieved (2/3), new leader elected

Data Loss:  NONE — committed entries replicated before failure

Why Healthy: 3-node cluster requires 2/3 quorum. Losing 1 = 2 remain = quorum maintained.

Mitigation:
  - Run odd-numbered etcd clusters (3, 5, 7 nodes)
  - Monitor RAFT TERM increments as election alerts
  - Set election-timeout based on network RTT between nodes
  - Use PodDisruptionBudgets to prevent simultaneous pod deletions
```

---

## Failure Mode 2: DNS Exclusion of NotReady Pods

This failure appears naturally after etcd-0 is deleted and restarted.

### Observable Symptoms
```bash
# Pod shows Running but Ready=False
k get pods | grep etcd-0

# DNS fails for etcd-0 despite pod running
docker exec k8s-net-lab-control-plane kubectl exec etcd-1 -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --endpoints=http://etcd-0.etcd:2379,http://etcd-1.etcd:2379,http://etcd-2.etcd:2379 \
  endpoint status -w table"
# etcd-0 times out with: "lookup etcd-0.etcd: no such host"

# Confirm pod is in NotReady
docker exec k8s-net-lab-control-plane kubectl describe endpoints etcd
# Look for: NotReadyAddresses: 10.244.x.x
```

### Why This Happens
```bash
# Kubernetes intentionally excludes NotReady pods from DNS
# NotReadyAddresses = pod is Running but readiness probe failing
# CoreDNS correctly refuses to return the IP
```

### Triage
```bash
# Check readiness status
docker exec k8s-net-lab-control-plane kubectl describe pod etcd-0 | grep -A 5 "Conditions"

# Check logs for why pod is not ready
docker exec k8s-net-lab-control-plane kubectl logs etcd-0 --tail=10
```

---

## Failure Mode 2 Documentation

```
FAILURE MODE 2: DNS Exclusion of NotReady Pods
───────────────────────────────────────────────
Trigger:     Pod Running but failing readiness check
Observable:
  - kubectl get pods: STATUS=Running, READY=0/1
  - DNS lookup: "no such host"
  - kubectl describe endpoints: IP in NotReadyAddresses
  - etcdctl: "context deadline exceeded" for that endpoint

Why This Happens:
  Kubernetes headless service DNS only returns IPs of Ready pods.
  NotReady = failed readiness probe = excluded from DNS intentionally.
  This is CORRECT behavior, not a DNS bug.

Common Mistake:
  Engineers assume "DNS broken" — actually pod health is broken.
  Fix the pod readiness issue, not the DNS.

Triage:
  1. kubectl describe pod — check Conditions section
  2. kubectl logs <pod> — find why readiness fails
  3. kubectl describe endpoints — confirm IP in NotReadyAddresses

Mitigation:
  - Implement meaningful readiness probes
  - Monitor Ready vs Running separately in alerting
  - Distinguish "pod exists" from "pod is healthy"
```

---

## Failure Mode 3: Bootstrap Conflict After Pod Restart

This appears when etcd-0 restarts and finds an existing data directory.

### Observable Symptoms
```bash
# Pod crash-loops with 5+ restarts
k get pods | grep etcd-0
# etcd-0   0/1   CrashLoopBackOff   5 (2m ago)

# Logs show the root cause
docker exec k8s-net-lab-control-plane kubectl logs etcd-0 --tail=5
# "member 69691ed70da97612 has already been bootstrapped"
# "failed to start etcd"
```

### Why This Happens
```
etcd-0 pod deleted → StatefulSet creates new pod
New pod starts with --initial-cluster-state=new
But /etcd-0.etcd/ data directory exists from previous run
etcd refuses: "already bootstrapped" = can't start
Pod crashes → restarts → crashes → CrashLoopBackOff
Pod stays NotReady → excluded from DNS → cluster degraded
```

### Recovery
```bash
# Full redeploy is required (data dir unreachable during crashloop)
docker exec k8s-net-lab-control-plane kubectl delete statefulset etcd --force --grace-period=0
docker exec k8s-net-lab-control-plane kubectl delete service etcd
docker exec k8s-net-lab-control-plane kubectl delete pods etcd-0 etcd-1 etcd-2 \
  --force --grace-period=0 2>/dev/null

sleep 10

docker cp etcd-cluster.yaml k8s-net-lab-control-plane:/etcd-cluster.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /etcd-cluster.yaml

k get pods -w | grep etcd
# Wait for all 3 Running
```

---

## Failure Mode 3 Documentation

```
FAILURE MODE 3: Bootstrap Conflict After Pod Restart
──────────────────────────────────────────────────────
Trigger:     StatefulSet pod restarts without PersistentVolumeClaims
Observable:
  - kubectl get pods: READY=0/1, RESTARTS=5+
  - kubectl logs: "member already bootstrapped"
  - kubectl describe pod: Ready=False
  - DNS: "no such host" for that pod

Root Cause:
  Container filesystem retains /etcd-X.etcd/ data directory.
  --initial-cluster-state=new conflicts with existing data.
  etcd refuses to bootstrap over existing member data.

Cascade:
  crash loop → NotReady → DNS exclusion → peer unreachable
  → cluster runs degraded on 2/3 nodes (still has quorum)

Recovery:
  Short term: delete StatefulSet and redeploy fresh
  Cannot delete data dir while pod is crash-looping

Production Fix:
  ALWAYS use PersistentVolumeClaims with etcd StatefulSets.
  Data lives outside the container — survives restarts cleanly.
  Use --initial-cluster-state=existing for members rejoining.

Prevention:
  - PVCs for all stateful workloads
  - PodDisruptionBudgets to prevent simultaneous pod restarts
  - Init containers to detect stale data and handle gracefully
  - Separate monitoring for Ready vs Running pod status
```

---

## Step 4: Verify Full Recovery

```bash
docker exec k8s-net-lab-control-plane kubectl exec etcd-0 -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --endpoints=http://etcd-0.etcd:2379,http://etcd-1.etcd:2379,http://etcd-2.etcd:2379 \
  endpoint status -w table"
# All 3 nodes, same RAFT TERM, same RAFT INDEX
```

---

## What You Learned

| Concept | Evidence |
|---------|----------|
| Raft leader election | Term incremented, new leader in <4ms |
| Zero data loss | Written key survived leadership change |
| DNS exclusion is correct | NotReady pods intentionally hidden from DNS |
| Bootstrap conflict cascade | Missing PVCs → crashloop → DNS exclusion |
| Recovery procedure | Delete StatefulSet, redeploy from scratch |
| Quorum math | 3-node cluster survives 1 failure (2/3 quorum) |

---

## Troubleshooting

See [issues.md](issues.md) for:
- `Issue 8` — etcd "already bootstrapped" crash loop
- `Issue 10` — kubectl exec fails on crash-looping pod

---

➡️ **Project Complete!** See [README.md](README.md) for the full summary.
