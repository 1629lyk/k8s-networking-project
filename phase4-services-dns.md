# Phase 4 — Services & DNS

> **Goal:** Understand how Kubernetes Services work under the hood — ClusterIP virtual IPs, kube-proxy iptables rules, CoreDNS resolution, and what DNS failure looks like at the packet level.  
> **Time:** ~3-4 hours  
> **Output:** DNS failure reproduced and diagnosed, iptables DNAT rules inspected

---

## The Problem Services Solve

Pods die and get new IPs constantly. If your frontend hardcodes `10.244.11.4` to reach the backend, it breaks every time the backend restarts.

**Services** give you a stable virtual IP (ClusterIP) and DNS name that always routes to healthy pods. But the ClusterIP doesn't exist on any machine — it lives only in iptables rules that rewrite packets on the fly.

```
Pod → ClusterIP:80 → iptables DNAT → real Pod IP:80
```

---

## Step 1: Deploy a Web Service

```bash
cat <<EOF > web-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

docker cp web-service.yaml k8s-net-lab-control-plane:/web-service.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /web-service.yaml
```

Wait for 3 replicas:
```bash
k get pods -o wide -w
# Wait for all 3 web-backend-xxx pods Running

k get service web-service
# Note the CLUSTER-IP — this is the virtual IP
```

---

## Step 2: Deploy a Client Pod

```bash
cat <<EOF > client-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    app: client
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

docker cp client-pod.yaml k8s-net-lab-control-plane:/client-pod.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /client-pod.yaml

docker exec k8s-net-lab-control-plane kubectl wait --for=condition=ready pod/client --timeout=60s
```

---

## Step 3: Prove ClusterIP is Virtual

```bash
CLUSTER_IP=$(docker exec k8s-net-lab-control-plane kubectl get svc web-service -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $CLUSTER_IP"

# Hit the ClusterIP via HTTP
docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s http://$CLUSTER_IP | grep -o "<title>.*</title>"
```

Expected: `<title>Welcome to nginx!</title>`

> The ClusterIP doesn't exist on any machine — it's just a destination that iptables intercepts and rewrites before the packet leaves the node.

---

## Step 4: See the iptables DNAT Rules

```bash
WORKER_PID=$(docker inspect k8s-net-lab-worker --format '{{.State.Pid}}')
CLUSTER_IP=$(docker exec k8s-net-lab-control-plane kubectl get svc web-service -o jsonpath='{.spec.clusterIP}')

# View the KUBE-SVC chain entry for web-service
sudo nsenter -t $WORKER_PID -n iptables -t nat -L KUBE-SERVICES -n | grep web

# View the load balancing rules
sudo nsenter -t $WORKER_PID -n iptables -t nat -L -n | grep -A 4 "$CLUSTER_IP"
```

Expected output:
```
KUBE-SVC-xxx  tcp  0.0.0.0/0  10.x.x.x  ← ClusterIP intercepted

KUBE-SEP-xxx  probability 0.33333333349  → 10.244.x.x:80  ← pod 1
KUBE-SEP-xxx  probability 0.50000000000  → 10.244.x.x:80  ← pod 2
KUBE-SEP-xxx  (remainder)                → 10.244.x.x:80  ← pod 3
```

> This is **probabilistic load balancing in iptables**.  
> 1/3 chance → pod 1, 50% of remaining 2/3 → pod 2, rest → pod 3 = exactly 33% each.

---

## Step 5: DNS Deep Dive

### Check CoreDNS is running
```bash
k get pods -n kube-system | grep coredns
```

### Resolve service by DNS name
```bash
docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s http://web-service | grep -o "<title>.*</title>"
```

### Full DNS trace with dig
```bash
docker exec k8s-net-lab-control-plane kubectl exec client -- dig web-service.default.svc.cluster.local
```

Look for:
```
;; ANSWER SECTION:
web-service.default.svc.cluster.local. 30 IN A 10.x.x.x   ← ClusterIP returned

;; SERVER: 10.96.0.10#53   ← CoreDNS answered
```

### View auto-injected DNS config
```bash
docker exec k8s-net-lab-control-plane kubectl exec client -- cat /etc/resolv.conf
```

Expected:
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

> Kubernetes auto-injects this into **every pod**.  
> `search` domains mean `curl http://web-service` expands to `web-service.default.svc.cluster.local` automatically.  
> `ndots:5` causes dual A+AAAA queries for every lookup — important for DNS load planning.

---

## Step 6: Simulate DNS Failure

### Add label to client pod (required for NetworkPolicy to match)
```bash
docker exec k8s-net-lab-control-plane kubectl label pod client app=client

# Verify
k get pod client --show-labels
```

### Block DNS traffic with NetworkPolicy
```bash
cat <<EOF > block-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-dns
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - port: 80
      protocol: TCP
EOF

docker cp block-dns.yaml k8s-net-lab-control-plane:/block-dns.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /block-dns.yaml
```

### Test the DNS failure signature
```bash
# DNS resolution — should time out
docker exec k8s-net-lab-control-plane kubectl exec client -- nslookup web-service
# Expected: timed out, no servers could be reached

# Curl by name — should fail
docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s --max-time 3 http://web-service \
  || echo "❌ DNS name failed"

# Curl by direct IP — should SUCCEED
CLUSTER_IP=$(docker exec k8s-net-lab-control-plane kubectl get svc web-service -o jsonpath='{.spec.clusterIP}')
docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s --max-time 3 http://$CLUSTER_IP \
  | grep title || echo "❌ direct IP also failed"
```

Expected results:
```
❌ DNS name failed       ← DNS is broken
<title>Welcome to nginx!</title>   ← direct IP works fine
```

> **This is the classic DNS failure signature.**  
> The service is healthy. The pods are healthy. The routing is healthy.  
> Only DNS is broken — but it makes the entire service unreachable by name.

### Restore DNS
```bash
docker exec k8s-net-lab-control-plane kubectl delete networkpolicy block-dns

# Verify DNS is back
docker exec k8s-net-lab-control-plane kubectl exec client -- nslookup web-service
docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s http://web-service | grep title
```

---

## DNS Failure Triage Decision Tree

```
App can't reach service
        │
        ├── curl by IP works?
        │     ├── YES → DNS problem
        │     │         ├── nslookup times out? → port 53 egress blocked
        │     │         └── nslookup returns wrong IP? → stale DNS cache
        │     └── NO  → Routing or NetworkPolicy problem
        │
        └── Check: k get pod --show-labels
                   k get networkpolicies
                   k describe endpoints <service>
```

---

## What You Learned

| Concept | Evidence |
|---------|----------|
| ClusterIP is virtual | Only exists in iptables, no machine owns it |
| kube-proxy load balancing | 33/33/33 probabilistic DNAT rules |
| CoreDNS auto-wired | /etc/resolv.conf injected into every pod |
| ndots:5 dual queries | Every lookup fires A + AAAA simultaneously |
| DNS failure signature | Name fails, direct IP works = DNS blocked |
| Label mismatch silent failure | Policy with wrong label = does nothing |

---

## Troubleshooting

See [issues.md](issues.md) for:
- `Issue 5` — CoreDNS crash-looping after fresh install
- `Issue 7` — NetworkPolicy silently does nothing (label mismatch)

---

➡️ **Next:** [Phase 5 — Packet Capture & conntrack](phase5-packet-capture.md)
