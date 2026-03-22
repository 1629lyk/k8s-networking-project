# Phase 5 — Packet Capture & conntrack

> **Goal:** Capture live traffic at the kernel level using tcpdump and nsenter. Inspect the connection tracking table with conntrack to see DNAT state pairs.  
> **Time:** ~3-4 hours  
> **Output:** Full HTTP lifecycle captured, DNS queries traced, conntrack DNAT pairs read

---

## Why This Phase Matters

Logs and metrics tell you *what* failed. Packet capture tells you *why*.

In production environments, nodes rarely have diagnostic tools installed. `nsenter` lets you run tools from your host machine inside any container's or pod's network namespace — the technique used by real SREs during live incidents.

---

## Prerequisites

The `client` pod and `web-service` from Phase 4 should still be running:

```bash
k get pods
k get service web-service
```

If not, redeploy:
```bash
docker cp web-service.yaml k8s-net-lab-control-plane:/web-service.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /web-service.yaml
docker cp client-pod.yaml k8s-net-lab-control-plane:/client-pod.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /client-pod.yaml
docker exec k8s-net-lab-control-plane kubectl wait --for=condition=ready pod/client --timeout=60s
```

---

## Step 1: Set Up Continuous Traffic

Open a **second terminal** and run this traffic loop:

```bash
# Terminal 2 — keep this running throughout the phase
while true; do
  docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s http://web-service > /dev/null \
    && echo "$(date): ok"
  sleep 1
done
```

All capture commands go in **Terminal 1**.

---

## Step 2: Get Node PID for nsenter

> `nsenter` requires the host-level PID of the target container/node to enter its network namespace.

```bash
# Get which node the client pod runs on
CLIENT_NODE=$(docker exec k8s-net-lab-control-plane kubectl get pod client -o jsonpath='{.spec.nodeName}')
echo "Client is on: $CLIENT_NODE"

# Get that node's PID
CLIENT_NODE_PID=$(docker inspect $CLIENT_NODE --format '{{.State.Pid}}')
echo "Node PID: $CLIENT_NODE_PID"

# Store for reuse
export CLIENT_NODE_PID
```

---

## Step 3: Capture HTTP Traffic

```bash
sudo nsenter -t $CLIENT_NODE_PID -n tcpdump -i any port 80 -n -c 20
```

### Reading the Output

```
cali* In  10.244.x.x:PORT > 10.x.x.x:80  [S]    ← SYN enters via pod's veth
eth0  Out 10.244.x.x:PORT > 10.244.y.y:80 [S]    ← iptables DNAT rewrites ClusterIP → pod IP
eth0  In  10.244.y.y:80 > 10.244.x.x:PORT [S.]   ← SYN-ACK from real pod
cali* Out 10.x.x.x:80 > 10.244.x.x:PORT   [S.]   ← iptables restores ClusterIP on return path
```

Key insight: **the same packet appears twice** — once with the ClusterIP destination, once with the real pod IP. That's the DNAT translation happening live.

Full HTTP lifecycle visible:
```
[S]    TCP SYN           ← connection start
[S.]   TCP SYN-ACK       ← server accepts
[.]    TCP ACK           ← handshake complete
[P.]   GET / HTTP/1.1    ← HTTP request (75 bytes)
[P.]   HTTP/1.1 200 OK   ← response headers
[P.]   HTTP body         ← nginx HTML (896 bytes)
[F.]   TCP FIN           ← client done
[F.]   TCP FIN           ← server done
```

---

## Step 4: Capture DNS Traffic

```bash
# Start DNS capture in background
sudo nsenter -t $CLIENT_NODE_PID -n tcpdump -i any port 53 -n -c 20 &

# Trigger a DNS lookup
docker exec k8s-net-lab-control-plane kubectl exec client -- nslookup web-service

wait
```

### Reading DNS Capture

```
cali* In  10.244.x.x > 10.96.0.10:53    A? web-service.default.svc.cluster.local
eth0  Out 10.244.x.x > 10.244.z.z:53    A? web-service.default.svc.cluster.local
                                          ↑ DNAT: CoreDNS ClusterIP → real CoreDNS pod IP

eth0  In  10.244.z.z:53 > 10.244.x.x    A 10.x.x.x (ClusterIP)
cali* Out 10.96.0.10:53 > 10.244.x.x    A 10.x.x.x (ClusterIP)
                                          ↑ DNAT restored on return path
```

> CoreDNS is itself a Kubernetes Service — its ClusterIP (`10.96.0.10`) gets DNAT'd to a real pod IP just like any other service.

> **ndots:5 in action:** Every lookup fires **two queries** — `A?` (IPv4) and `AAAA?` (IPv6) simultaneously. At scale with thousands of pods this DNS traffic adds up significantly.

---

## Step 5: Inspect the conntrack Table

```bash
CLUSTER_IP=$(docker exec k8s-net-lab-control-plane kubectl get svc web-service -o jsonpath='{.spec.clusterIP}')

# View HTTP connection tracking entries
sudo nsenter -t $CLIENT_NODE_PID -n conntrack -L 2>/dev/null | grep "$CLUSTER_IP" | head -10
```

### Reading conntrack Output

```
tcp 6 77 TIME_WAIT
  src=10.244.x.x dst=10.96.y.y sport=46566 dport=80    ← original: pod→ClusterIP
  src=10.244.z.z dst=10.244.x.x sport=80 dport=46566   ← translated: real pod→client
  [ASSURED]
```

Each entry stores **both directions as a single record**:
- Original connection: pod → ClusterIP
- Translated connection: real pod → client

When the real pod sends a reply, the kernel looks up this record and rewrites the source IP back to the ClusterIP before the client pod sees it. The client never knows the real pod IP.

```bash
# View DNS connection tracking (UDP)
sudo nsenter -t $CLIENT_NODE_PID -n conntrack -L 2>/dev/null | grep "dport=53" | head -5

# Total connection count
sudo nsenter -t $CLIENT_NODE_PID -n conntrack -C 2>/dev/null
```

> **Production warning:** When `conntrack -C` reaches `nf_conntrack_max` (default ~262144), all new connections are silently dropped. This is one of the nastiest Kubernetes failure modes — no error message, just dropped connections.

---

## Step 6: Trace a Single Connection End-to-End

Put it all together — trace one complete HTTP request from DNS lookup to TCP close:

```bash
# Capture everything at once
sudo nsenter -t $CLIENT_NODE_PID -n tcpdump -i any "(port 80 or port 53)" -n -c 40 &

# Make one request
docker exec k8s-net-lab-control-plane kubectl exec client -- curl -s http://web-service > /dev/null

wait
```

You'll see the complete sequence:
1. `A? web-service...` — DNS query
2. `A 10.x.x.x` — DNS answer (ClusterIP)
3. `[S]` to ClusterIP — TCP SYN
4. DNAT → real pod IP
5. `[S.]` from real pod — SYN-ACK
6. DNAT restored → ClusterIP in reply
7. `GET / HTTP/1.1`
8. `HTTP/1.1 200 OK`
9. `[F.]` — connection close

---

## What You Learned

| Skill | Evidence |
|-------|----------|
| nsenter technique | Captured packets without tools on node |
| DNAT translation visible | Packet appears twice: ClusterIP then real IP |
| Full TCP lifecycle | SYN→data→FIN all in one capture |
| DNS uses same DNAT | CoreDNS is just another Service |
| ndots:5 dual queries | A + AAAA fires for every single lookup |
| conntrack DNAT pairs | Original + translated IPs in same entry |
| conntrack exhaustion risk | Total count visible, max limit documented |

---

## Quick Reference: nsenter Patterns

```bash
# Enter a node's network namespace
NODE_PID=$(docker inspect <node-name> --format '{{.State.Pid}}')
sudo nsenter -t $NODE_PID -n <command>

# Common captures
sudo nsenter -t $NODE_PID -n tcpdump -i any -n             # all traffic
sudo nsenter -t $NODE_PID -n tcpdump -i any port 80 -n     # HTTP only
sudo nsenter -t $NODE_PID -n tcpdump -i any port 53 -n     # DNS only
sudo nsenter -t $NODE_PID -n tcpdump -i eth0 icmp -n       # ICMP only
sudo nsenter -t $NODE_PID -n tcpdump -i any -n -c 20       # stop after 20 packets

# conntrack
sudo nsenter -t $NODE_PID -n conntrack -L 2>/dev/null       # list all
sudo nsenter -t $NODE_PID -n conntrack -C 2>/dev/null       # count
sudo nsenter -t $NODE_PID -n conntrack -L 2>/dev/null | grep <IP>  # filter by IP

# iptables
sudo nsenter -t $NODE_PID -n iptables -t nat -L -n          # NAT rules
sudo nsenter -t $NODE_PID -n iptables -t nat -L KUBE-SERVICES -n  # Service entries
```

---

## Troubleshooting

See [issues.md](issues.md) for:
- `Issue 6` — tcpdump not found on kind node (use nsenter from host)

---

➡️ **Next:** [Phase 6 — Observability with Goldpinger](phase6-observability.md)
