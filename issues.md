# Known Issues & Fixes

> All issues encountered during this project on Ubuntu 22.04 WSL2 with kind v0.31.0.  
> Each issue includes exact symptoms, root cause, and the working fix.

---

## Issue 1: kind Cluster Fails — "kubelet unhealthy" / cgroup Error

**Phase:** Phase 1  
**Symptom:**
```
✗ Starting control-plane
ERROR: failed to create cluster: failed to init node with kubeadm
The kubelet is not healthy after 4m0s
The kubelet is unhealthy due to a misconfiguration (required cgroups disabled)
```

**Root Cause:**  
The default `kindest/node` image pulled by kind v0.31.0 is `v1.35.0` which uses cgroup features not supported by the WSL2 kernel.

**Fix:**  
Always pin to a tested, WSL2-compatible node image with a SHA hash:

```bash
# In kind-cluster.yaml, always use this exact image reference:
nodes:
  - role: control-plane
    image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
  - role: worker
    image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
  - role: worker
    image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
```

```bash
# Clean up failed cluster before retrying
kind delete cluster --name k8s-net-lab
kind create cluster --config kind-cluster.yaml
```

---

## Issue 2: kubectl TLS Timeout on WSL2

**Phase:** Phase 1 (affects all phases)  
**Symptom:**
```
Unable to connect to the server: net/http: TLS handshake timeout
```
or
```
dial tcp 172.18.0.3:6443: i/o timeout
```

**Root Cause:**  
WSL2 runs inside a VM with its own network stack. The Docker bridge (`172.18.0.0/16`) is not directly reachable from WSL2 userspace. Port forwarding (`127.0.0.1:PORT → container:6443`) also fails intermittently.

**Fix:**  
Use `docker exec` as a kubectl proxy — this runs kubectl inside the control-plane container where network access is direct:

```bash
echo 'alias k="docker exec k8s-net-lab-control-plane kubectl"' >> ~/.bashrc
source ~/.bashrc

# Verify
k get nodes
```

> ⚠️ Use `k` instead of `kubectl` for **all** commands throughout this project.

**Note:** If the alias gets lost between sessions:
```bash
source ~/.bashrc
# or re-add it
echo 'alias k="docker exec k8s-net-lab-control-plane kubectl"' >> ~/.bashrc
source ~/.bashrc
```

---

## Issue 3: Another Networking Lab (ContainerLab) Interfering

**Phase:** Phase 1, Phase 6  
**Symptom:**
```
Unable to connect to the server: net/http: TLS handshake timeout
```
Docker containers seem healthy but kubectl is completely unreachable. Cluster was working before.

**Root Cause:**  
ContainerLab (`clab-*`) containers create their own Docker bridge networks and routing rules that conflict with kind's `172.18.0.0/16` network, causing packet routing failures.

**Fix:**
```bash
# Stop all clab containers
docker stop $(docker ps -q --filter "name=clab") 2>/dev/null

# Remove clab networks
docker network ls --format "{{.Name}}" | grep clab | xargs -r docker network rm

# Prune any orphaned networks
docker network prune -f

# Verify nothing is interfering
docker ps | grep clab
docker network ls | grep clab
# Both should return nothing

# Wait 10 seconds for routing to settle
sleep 10
k get nodes
```

> **Prevention:** Always stop ContainerLab labs before working with kind clusters. Running two separate network labs simultaneously on WSL2 causes unpredictable routing conflicts.

---

## Issue 4: EOF Piping Fails with k Alias

**Phase:** Phase 2 (affects all phases that apply manifests)  
**Symptom:**
```
error: no objects passed to apply
```

**Root Cause:**  
The `k` alias (`docker exec k8s-net-lab-control-plane kubectl`) doesn't inherit stdin from the shell. `cat <<EOF | k apply -f -` pipes to docker exec but the data never reaches kubectl.

**Fix:**  
Always save manifests to files and use `docker cp` to transfer them:

```bash
# WRONG — won't work with the alias
cat <<EOF | k apply -f -
apiVersion: v1
...
EOF

# CORRECT — always do this instead
cat <<EOF > manifest.yaml
apiVersion: v1
...
EOF

docker cp manifest.yaml k8s-net-lab-control-plane:/manifest.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /manifest.yaml
```

**Alternative** (for simple inline manifests):
```bash
docker exec k8s-net-lab-control-plane bash -c "cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
...
EOF"
```

> Note: The `bash -c` approach works because the heredoc is evaluated inside the container.

---

## Issue 5: CoreDNS Crash-Looping After Fresh Install

**Phase:** Phase 2, Phase 6  
**Symptom:**
```bash
k get pods -n kube-system | grep coredns
# coredns-xxx   0/1   Running   5 (41s ago)
```

Logs show:
```
dial tcp 10.96.0.1:443: i/o timeout
Failed to watch *v1.Service: context deadline exceeded
```

**Root Cause:**  
CoreDNS tries to connect to the Kubernetes API server via its ClusterIP (`10.96.0.1`). If the CNI hasn't fully programmed eBPF maps or iptables rules for that service yet, CoreDNS can't reach the API server and crash-loops.

**Fix:**
```bash
# Step 1: Ensure CNI is fully healthy first
k get pods -n calico-system
# ALL must show 1/1 Running before proceeding

# Step 2: Restart CoreDNS
docker exec k8s-net-lab-control-plane kubectl rollout restart deployment/coredns -n kube-system

# Step 3: Watch recovery
k get pods -n kube-system -w | grep coredns
# Wait for 1/1 Running
```

> If CoreDNS keeps crash-looping even after Calico is healthy, restart the entire Calico node daemonset:
> ```bash
> docker exec k8s-net-lab-control-plane kubectl rollout restart daemonset/calico-node -n calico-system
> sleep 30
> docker exec k8s-net-lab-control-plane kubectl rollout restart deployment/coredns -n kube-system
> ```

---

## Issue 6: tcpdump Not Found on kind Node

**Phase:** Phase 5  
**Symptom:**
```
OCI runtime exec failed: exec failed: unable to start container process: exec: "tcpdump": executable file not found in $PATH
```

**Root Cause:**  
kind node images are minimal — they don't include diagnostic tools like `tcpdump`, `ping`, `route`, etc.

**Fix:**  
Use `nsenter` to run tools from the WSL2 host inside the node's network namespace:

```bash
# Get the host-level PID of the kind node container
NODE_PID=$(docker inspect k8s-net-lab-worker --format '{{.State.Pid}}')
echo "Node PID: $NODE_PID"

# Run tcpdump from host inside node's network namespace
sudo nsenter -t $NODE_PID -n tcpdump -i eth0 -n

# Other tools that work the same way
sudo nsenter -t $NODE_PID -n conntrack -L 2>/dev/null
sudo nsenter -t $NODE_PID -n iptables -t nat -L -n
sudo nsenter -t $NODE_PID -n ip route
```

> `nsenter -n` enters only the network namespace. The tools (tcpdump, conntrack, etc.) run from your WSL2 host but see the container's network interfaces.

---

## Issue 7: NetworkPolicy Silently Does Nothing

**Phase:** Phase 3, Phase 4  
**Symptom:**  
Applied a NetworkPolicy but traffic is not blocked. Policy exists in `k get networkpolicies` but has no effect.

**Root Cause:**  
The `podSelector` in the NetworkPolicy references labels that don't exist on the target pods. The policy applies to zero pods, so nothing changes.

**Diagnosis:**
```bash
# Check what labels pods actually have
k get pods --show-labels

# Compare with what the policy expects
k describe networkpolicy <policy-name>
# Look at "PodSelector:" line
```

**Fix:**
```bash
# Option 1: Add the missing label to the pod
docker exec k8s-net-lab-control-plane kubectl label pod <pod-name> <key>=<value>

# Option 2: Fix the policy's podSelector to match existing labels

# Verify fix
k get pod <pod-name> --show-labels
# Confirm the label matches what the policy expects
```

> **Prevention:** Always run `k get pods --show-labels` before writing NetworkPolicy.  
> The most common mistake is writing `app: client` when the pod has no labels at all.

---

## Issue 8: etcd "Already Bootstrapped" Crash Loop

**Phase:** Phase 7  
**Symptom:**
```bash
k get pods | grep etcd
# etcd-0   0/1   CrashLoopBackOff   5 (2m ago)

docker exec k8s-net-lab-control-plane kubectl logs etcd-0 --tail=5
# "member 69691ed70da97612 has already been bootstrapped"
# "failed to start etcd"
```

**Root Cause:**  
The etcd StatefulSet has no PersistentVolumeClaims (PVCs). The data directory (`/etcd-0.etcd/`) lives in the container's filesystem and survives container restarts. When the pod restarts, etcd tries to bootstrap with `--initial-cluster-state=new` but finds existing data and refuses to start.

**Fix (Full Redeploy):**
```bash
# Cannot access the data dir while pod is crash-looping
# Full redeploy is the only option

docker exec k8s-net-lab-control-plane kubectl delete statefulset etcd --force --grace-period=0
docker exec k8s-net-lab-control-plane kubectl delete service etcd
docker exec k8s-net-lab-control-plane kubectl delete pods etcd-0 etcd-1 etcd-2 \
  --force --grace-period=0 2>/dev/null

# Wait for cleanup
sleep 10

# Redeploy fresh
docker cp etcd-cluster.yaml k8s-net-lab-control-plane:/etcd-cluster.yaml
docker exec k8s-net-lab-control-plane kubectl apply -f /etcd-cluster.yaml

k get pods -w | grep etcd
```

**Production Prevention:**  
Always use PersistentVolumeClaims with etcd:
```yaml
volumeClaimTemplates:
- metadata:
    name: etcd-data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 1Gi
```

---

## Issue 9: Stale kubeconfig After Cluster Recreate

**Phase:** Phase 1, any phase where cluster is recreated  
**Symptom:**  
kubectl was working, cluster was deleted and recreated, now kubectl fails with TLS errors or connection refused.

**Root Cause:**  
When kind recreates a cluster, it assigns a new port for the API server (e.g., `40845` instead of `33755`). The kubeconfig still points to the old port.

**Fix:**
```bash
# Option 1: Re-export kubeconfig
kind export kubeconfig --name k8s-net-lab

# Option 2: Use the docker exec alias (more reliable on WSL2)
echo 'alias k="docker exec k8s-net-lab-control-plane kubectl"' >> ~/.bashrc
source ~/.bashrc
k get nodes
```

> On WSL2, the docker exec alias is always more reliable than kubeconfig-based kubectl because it bypasses the port forwarding entirely.

---

## Issue 10: kubectl exec Fails on Crash-Looping Pod

**Phase:** Phase 7  
**Symptom:**
```
error: unable to upgrade connection: container not found ("etcd")
```

**Root Cause:**  
The pod is in CrashLoopBackOff — the container starts, immediately crashes, and kubectl exec tries to connect between restarts when no container is running.

**Fix Option A: Catch it during brief running window**
```bash
# Loop until exec succeeds
for i in $(seq 1 20); do
  docker exec k8s-net-lab-control-plane kubectl exec <pod-name> -- <command> 2>/dev/null && break
  sleep 2
done
```

**Fix Option B: Access via crictl on the node**
```bash
# Find which node the pod is on
NODE=$(docker exec k8s-net-lab-control-plane kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}')

# List containers (including exited ones)
docker exec $NODE crictl ps -a | grep <pod-name>

# Get container ID and exec into it while running
CONTAINER_ID=$(docker exec $NODE crictl ps -a | grep <pod-name> | head -1 | awk '{print $1}')
docker exec $NODE crictl exec $CONTAINER_ID <command>
```

**Fix Option C: For etcd specifically — full redeploy**  
See Issue 8. If etcd is crash-looping due to bootstrap conflict, `kubectl exec` will never succeed. Redeploy is required.

---

## Quick Diagnostics Cheat Sheet

```bash
# Is the cluster reachable?
k get nodes

# Are all pods healthy?
k get pods -A

# What's wrong with a specific pod?
k describe pod <pod-name>
k logs <pod-name> --tail=20
k logs <pod-name> --previous  # logs from last crash

# Are NetworkPolicies matching anything?
k get pods --show-labels
k get networkpolicies
k describe networkpolicy <name>

# Are service endpoints populated?
k get endpoints <service-name>
k describe endpoints <service-name>
# NotReadyAddresses means pod is Running but not Ready

# Is DNS working from inside a pod?
docker exec k8s-net-lab-control-plane kubectl exec <pod> -- nslookup kubernetes
docker exec k8s-net-lab-control-plane kubectl exec <pod> -- cat /etc/resolv.conf

# Is there network traffic flowing?
NODE_PID=$(docker inspect k8s-net-lab-worker --format '{{.State.Pid}}')
sudo nsenter -t $NODE_PID -n tcpdump -i any -n -c 20

# Is conntrack table filling up?
NODE_PID=$(docker inspect k8s-net-lab-worker --format '{{.State.Pid}}')
sudo nsenter -t $NODE_PID -n conntrack -C 2>/dev/null
```
