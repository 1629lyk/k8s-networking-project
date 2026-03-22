# Phase 2 — Calico CNI

> **Goal:** Install Calico as the cluster CNI, understand how pod IPs are assigned and routed, and verify cross-node pod communication at the packet level.  
> **Time:** ~4-6 hours  
> **Output:** All nodes `Ready`, pods communicating across nodes, routing tables inspected

---

## What is a CNI?

When Kubernetes creates a pod, something must:
1. Give it an IP address
2. Create a virtual network interface
3. Wire it up so other pods can reach it

That "something" is the CNI plugin. Without it, nodes stay `NotReady` and no pods can be scheduled.

**Calico** uses BGP routing and programs `iptables`/`ip route` rules on every node so packets can flow between pods on different nodes without any tunneling overhead.

---

## Pre-Check

```bash
k get nodes
# All should show NotReady — correct, no CNI yet
```

> ⚠️ **WSL2 Reminder:** Use `k` (the alias) not `kubectl`.  
> For manifests, always save to a file and `docker cp` — piping via EOF doesn't work with the alias.

---

## Step 1: Install the Calico Operator

```bash
k create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```

Wait ~10 seconds for the operator to start.

---

## Step 2: Configure Calico's IP Pool

```bash
cat <<EOF > calico-install.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
EOF

docker cp calico-install.yaml k8s-net-lab-control-plane:/calico-install.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /calico-install.yaml
```

> The `cidr: 10.244.0.0/16` matches the `podSubnet` in `kind-cluster.yaml` exactly.  
> `blockSize: 26` means each node gets a `/26` subnet (64 IPs) from the pool.  
> `VXLANCrossSubnet` encapsulates packets that cross subnet boundaries.

---

## Step 3: Watch Calico Come Online

Open a **second terminal** and watch nodes flip to Ready:

```bash
# Terminal 2
k get nodes -w
```

Watch Calico pods in **Terminal 1**:

```bash
k get pods -n calico-system -w
```

Wait until all pods show `1/1 Running` (~2-3 minutes):

```
calico-kube-controllers-xxx   1/1 Running
calico-node-xxx               1/1 Running   (one per node)
calico-node-xxx               1/1 Running
calico-node-xxx               1/1 Running
calico-typha-xxx              1/1 Running
```

### Verify nodes are Ready
```bash
k get nodes
```

Expected:
```
NAME                        STATUS   ROLES           AGE
k8s-net-lab-control-plane   Ready    control-plane   Xm
k8s-net-lab-worker          Ready    <none>          Xm
k8s-net-lab-worker2         Ready    <none>          Xm
```

---

## Step 4: Inspect What Calico Built

### View routing table on control-plane
```bash
docker exec k8s-net-lab-control-plane ip route
```

Sample output:
```
blackhole 10.244.139.128/26 proto 80          ← "I own this subnet"
10.244.139.129 dev cali512f322e769             ← pod lives on this veth
10.244.11.0/26 via 172.18.0.7 dev eth0        ← worker2's pods are over there
10.244.134.64/26 via 172.18.0.6 dev eth0      ← worker's pods are over there
```

### View routing table on worker
```bash
docker exec k8s-net-lab-worker ip route
```

### View cali* veth interfaces
```bash
docker exec k8s-net-lab-control-plane ip link show type veth
```

> `cali*` interfaces are virtual ethernet pairs — one end is in the pod's network namespace, the other is on the node. This is how pods get network access.

---

## Step 5: Deploy Test Pods on Different Nodes

```bash
cat <<EOF > test-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
spec:
  nodeName: k8s-net-lab-worker
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
spec:
  nodeName: k8s-net-lab-worker2
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

docker cp test-pods.yaml k8s-net-lab-control-plane:/test-pods.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /test-pods.yaml
```

Wait for both pods and note their IPs:
```bash
k get pods -o wide -w
```

Expected:
```
NAME    READY   STATUS    IP              NODE
pod-a   1/1     Running   10.244.134.70   k8s-net-lab-worker
pod-b   1/1     Running   10.244.11.2     k8s-net-lab-worker2
```

> Each pod gets an IP from its node's `/26` block — proving Calico's per-node subnet assignment.

---

## Step 6: Test Cross-Node Communication

```bash
POD_B_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod pod-b -o jsonpath='{.status.podIP}')
POD_A_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod pod-a -o jsonpath='{.status.podIP}')

# Ping pod-b from pod-a (crosses nodes)
docker exec k8s-net-lab-control-plane kubectl exec pod-a -- ping -c 4 $POD_B_IP
```

Expected:
```
64 bytes from 10.244.11.2: icmp_seq=1 ttl=62 time=0.258 ms
0% packet loss
```

> `ttl=62` means the packet decremented from 64 by 2 — it crossed 2 node hops. Proof of cross-node routing.

---

## Step 7: Capture the Packets Live

Get the worker node's PID (for nsenter):
```bash
WORKER_PID=$(docker inspect k8s-net-lab-worker --format '{{.State.Pid}}')
echo "Worker PID: $WORKER_PID"
```

**Terminal 2** — start capture:
```bash
sudo nsenter -t $WORKER_PID -n tcpdump -i eth0 icmp -n
```

**Terminal 1** — send ping while capture runs:
```bash
POD_B_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod pod-b -o jsonpath='{.status.podIP}')
POD_A_IP=$(docker exec k8s-net-lab-control-plane kubectl get pod pod-a -o jsonpath='{.status.podIP}')
docker exec k8s-net-lab-control-plane kubectl exec pod-a -- ping -c 4 $POD_B_IP
```

Expected capture:
```
10.244.134.70 > 10.244.11.2: ICMP echo request   ← pod-a → pod-b
10.244.11.2 > 10.244.134.70: ICMP echo reply     ← pod-b → pod-a
```

> Pod IPs appear as-is (no NAT between pods). Calico routes packets natively.

---

## What You Learned

| Concept | Evidence |
|---------|----------|
| Calico assigns per-node subnets | Pod IPs from different /26 blocks per node |
| Routing table programmed by Calico | `ip route` shows `proto 80` (Calico) entries |
| veth pairs connect pods to node | `cali*` interfaces visible on node |
| Cross-node routing works natively | 0% packet loss ping across nodes |
| Packets use pod IPs (no NAT) | tcpdump shows real pod IPs |

---

## Troubleshooting

See [issues.md](issues.md) for:
- `Issue 4` — EOF piping with k alias fails (use docker cp instead)

---

➡️ **Next:** [Phase 3 — NetworkPolicy](phase3-networkpolicy.md)
