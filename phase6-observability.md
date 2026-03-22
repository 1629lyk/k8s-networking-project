# Phase 6 — Observability with Goldpinger

> **Goal:** Deploy a continuous cross-node connectivity monitor. Understand eBPF as an architectural concept and how it differs from iptables-based networking.  
> **Time:** ~2-3 hours  
> **Output:** Goldpinger running with live cross-node health data

---

## eBPF vs iptables — The Conceptual Shift

This phase originally aimed to deploy Cilium (eBPF-based CNI) on WSL2. After extensive testing, WSL2's kubelet port restrictions make Cilium unstable in this environment. Instead, we deploy **Goldpinger** on our Calico cluster and cover eBPF concepts explicitly.

### iptables (Calico) — What You've Been Using
```
packet arrives
  → traverses chain of iptables rules (can be thousands)
  → each rule evaluated sequentially
  → matching rule takes action (ACCEPT/DROP/DNAT)
  → packet leaves
```

### eBPF (Cilium) — The Modern Approach
```
packet arrives
  → eBPF program compiled into kernel bytecode runs immediately
  → no chain traversal — direct decision at wire speed
  → program can observe, modify, or drop the packet
  → packet leaves (or doesn't)
```

Key differences:

| | iptables (Calico) | eBPF (Cilium) |
|--|--|--|
| Rule evaluation | Sequential | Direct bytecode |
| Performance | Degrades with rule count | Constant time |
| Observability | Limited | Full flow visibility |
| kube-proxy | Required | Replaced entirely |
| WSL2 support | ✅ Stable | ⚠️ Limited |

> **In production:** Cilium is increasingly the preferred CNI for large clusters. The concepts are the same — this phase gives you the vocabulary and architecture understanding.

---

## Step 1: Deploy Goldpinger RBAC

Goldpinger needs permission to list pods via the Kubernetes API to discover its peers:

```bash
cat <<EOF > goldpinger-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: goldpinger
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: goldpinger
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "watch", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: goldpinger
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: goldpinger
subjects:
- kind: ServiceAccount
  name: goldpinger
  namespace: default
EOF

docker cp goldpinger-rbac.yaml k8s-net-lab-control-plane:/goldpinger-rbac.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /goldpinger-rbac.yaml
```

---

## Step 2: Deploy Goldpinger DaemonSet

A DaemonSet deploys one pod per node automatically — perfect for a network health monitor:

```bash
cat <<EOF > goldpinger.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: goldpinger
  namespace: default
spec:
  selector:
    matchLabels:
      app: goldpinger
  template:
    metadata:
      labels:
        app: goldpinger
    spec:
      serviceAccountName: goldpinger
      containers:
      - name: goldpinger
        image: bloomberg/goldpinger:3.10.0
        env:
        - name: HOST
          value: "0.0.0.0"
        - name: PORT
          value: "8080"
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: goldpinger
  namespace: default
spec:
  selector:
    app: goldpinger
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
EOF

docker cp goldpinger.yaml k8s-net-lab-control-plane:/goldpinger.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /goldpinger.yaml
```

Wait for pods:
```bash
k get pods -o wide -w | grep goldpinger
# One pod per worker node
```

---

## Step 3: Verify Cross-Node Health

```bash
GOLDPINGER_IP=$(docker exec k8s-net-lab-control-plane kubectl get svc goldpinger -o jsonpath='{.spec.clusterIP}')
echo "Goldpinger ClusterIP: $GOLDPINGER_IP"

docker exec k8s-net-lab-control-plane curl -s http://$GOLDPINGER_IP:8080/check_all | python3 -m json.tool
```

### Reading the Output

Healthy cluster:
```json
{
  "hosts": [
    { "hostIP": "172.18.0.2", "podIP": "10.244.x.x", "podName": "goldpinger-xxx" },
    { "hostIP": "172.18.0.3", "podIP": "10.244.y.y", "podName": "goldpinger-yyy" }
  ],
  "responses": {
    "goldpinger-xxx": {
      "OK": true,
      "response": {
        "podResults": {
          "goldpinger-yyy": {
            "OK": true,
            "response-time-ms": 1,
            "status-code": 200
          }
        }
      }
    }
  }
}
```

What each field means:

| Field | Meaning |
|-------|---------|
| `"OK": true` | Cross-node pod networking is healthy |
| `response-time-ms: 1` | Sub-millisecond latency — Calico VXLAN overhead is negligible |
| `hostIP` | Which node (Docker bridge IP) the pod runs on |
| `podIP` | Pod's Calico-assigned IP |
| `status-code: 200` | HTTP reachability confirmed |

---

## Step 4: Check Service Endpoints

```bash
docker exec k8s-net-lab-control-plane kubectl get endpoints goldpinger
```

Both goldpinger pod IPs should be listed. If empty or missing one, check pod labels:

```bash
k get pods -l app=goldpinger -o wide
k describe endpoints goldpinger
```

---

## Step 5: Simulate a Network Partition

Isolate one goldpinger pod with a NetworkPolicy and watch the health check reflect it:

```bash
# Get the name of one goldpinger pod
GP_POD=$(docker exec k8s-net-lab-control-plane kubectl get pods -l app=goldpinger -o jsonpath='{.items[0].metadata.name}')
echo "Isolating: $GP_POD"

cat <<EOF > isolate-goldpinger.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-goldpinger
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: goldpinger
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
EOF

docker cp isolate-goldpinger.yaml k8s-net-lab-control-plane:/isolate-goldpinger.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /isolate-goldpinger.yaml

# Wait 10 seconds then check health
sleep 10
docker exec k8s-net-lab-control-plane curl -s http://$GOLDPINGER_IP:8080/check_all | python3 -m json.tool | grep -E '"OK"|"response-time"'
```

Expected: some `"OK": false` entries — the isolated pod can't reach others.

### Restore
```bash
docker exec k8s-net-lab-control-plane kubectl delete networkpolicy isolate-goldpinger
sleep 5
docker exec k8s-net-lab-control-plane curl -s http://$GOLDPINGER_IP:8080/check_all | python3 -m json.tool | grep '"OK"'
# All should be "OK": true again
```

---

## What You Learned

| Concept | Evidence |
|---------|----------|
| eBPF vs iptables architecture | Conceptual — kernel bytecode vs rule chains |
| DaemonSet pattern | One pod per node, auto-scheduled |
| RBAC for pod discovery | ServiceAccount + ClusterRole required |
| Continuous health monitoring | Goldpinger shows real-time cross-node status |
| NetworkPolicy as chaos tool | Partition a pod and observe health impact |

---

## Goldpinger in Production

In real clusters, Goldpinger is used to:
- Alert when `OK: false` appears (network partition detected)
- Track latency trends between nodes (spikes indicate network congestion)
- Validate CNI is healthy after upgrades or node replacements
- Confirm cross-AZ connectivity in multi-zone clusters

The `/check_all` endpoint can feed directly into Prometheus + Grafana for dashboards.

---

## Troubleshooting

If `"hosts": null` in the check_all response:
```bash
# Goldpinger can't discover peers — RBAC issue
k get pods -l app=goldpinger --show-labels

# Verify ServiceAccount is attached
k describe pod <goldpinger-pod> | grep "Service Account"

# Check ClusterRoleBinding exists
k get clusterrolebinding goldpinger
```

---

➡️ **Next:** [Phase 7 — Raft Cluster & Failure Modes](phase7-raft-failures.md)
