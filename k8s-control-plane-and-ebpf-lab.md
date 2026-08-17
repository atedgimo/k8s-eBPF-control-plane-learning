# Hands-on Guide: Kubernetes Control Plane + eBPF

> Level: intermediate · Style: runnable labs · Assumes a Linux environment (Ubuntu/Debian recommended, or WSL2).
> Every command marked with `$` can be copied and run. If you don't have an environment yet, setup instructions are at the end.

## What we'll learn and how this doc is organized

The two topics are more connected than they seem: the **control plane** is the "brain" of Kubernetes that decides what should run and where, and **eBPF** is the technology that lets the cluster's networking, security, and observability run directly in the kernel without paying the performance cost of older approaches. At the end we tie them together through Cilium.

Structure:

1. **Part A — Control Plane**: every component, its job, and how to *watch* it work. 7 labs.
2. **Part B — eBPF**: the execution model, the verifier, maps, program types, and tooling. 6 labs.
3. **Part C — The connection**: how eBPF replaces kube-proxy and provides in-cluster observability.

---

# Part A — Kubernetes Control Plane

## 1.1 The big picture

A Kubernetes cluster is split into two planes:

- **Control Plane** — makes decisions: what is the desired state, and how to reach it.
- **Data Plane / Worker Nodes** — actually runs the containers (kubelet + container runtime + kube-proxy/CNI).

The core control-plane components:

| Component | Job, in brief | Stateful? |
|-----------|---------------|-----------|
| **kube-apiserver** | The single gateway. Every read/write goes through it. REST over HTTP. | No (stateless) |
| **etcd** | The database. Holds *all* cluster state. The source of truth. | Yes |
| **kube-scheduler** | Decides which node each new Pod goes to. | No |
| **kube-controller-manager** | A collection of "controllers" that run reconciliation loops. | No |
| **cloud-controller-manager** | Integration with the cloud provider (load balancers, volumes, nodes). | No |

**The overarching principle to internalize:** everything in Kubernetes is **declarative** + **level-triggered**. You declare a *desired state*, and each controller runs an infinite loop that compares desired state against observed state and acts to shrink the gap. This is the **reconciliation loop**. There are no one-shot "commands" — there's continuous convergence toward the desired state.

```
                    ┌─────────────┐
   kubectl ───────► │ kube-apiserver│ ◄──── kubelet (from every node)
                    └──────┬──────┘
                           │ (the only component that talks to etcd)
                    ┌──────▼──────┐
                    │    etcd     │  ← source of truth
                    └─────────────┘
                           ▲
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ┌─────┴─────┐    ┌───────┴──────┐   ┌───────┴────────┐
  │ scheduler │    │ controller-  │   │ cloud-         │
  │           │    │ manager      │   │ controller-mgr │
  └───────────┘    └──────────────┘   └────────────────┘
     all talk only through the apiserver, never directly to etcd
```

Note the critical point: **only the apiserver accesses etcd.** Everything else (scheduler, controllers, kubelet) talks solely to the apiserver via the API. This is what makes the architecture modular — you can add controllers and operators without ever touching etcd.

## Lab 0 — Spin up a local cluster

We'll use `kind` (Kubernetes IN Docker) because it runs the control plane as real components inside containers, which lets us "go inside" and observe them — unlike a managed cloud cluster where the control plane is hidden.

```bash
# Requires Docker installed and running
# Install kind
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
$ chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Install kubectl
$ curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Create a cluster with a control-plane + 2 workers
$ cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

$ kind create cluster --name lab --config kind-config.yaml
$ kubectl cluster-info --context kind-lab
```

## Lab 1 — See the control plane with your own eyes

In kubeadm/kind, the control-plane components run as **static pods**: the kubelet on the control-plane node reads YAML files from `/etc/kubernetes/manifests/` and runs them directly, without the scheduler. This solves a chicken-and-egg problem (how do you schedule the scheduler itself?).

```bash
# Control-plane components run as pods in the system namespace
$ kubectl get pods -n kube-system

# You'll see: kube-apiserver-lab-control-plane, etcd-lab-control-plane,
#             kube-scheduler-..., kube-controller-manager-...

# Go inside the control-plane node and look at the static manifests
$ docker exec -it lab-control-plane bash
root@lab-control-plane:/# ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

# Look at the flags the apiserver starts with — you learn a lot here
root@lab-control-plane:/# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -A40 'command:'
```

**Thought exercise:** in the apiserver file you'll see flags like `--etcd-servers`, `--service-cluster-ip-range`, `--authorization-mode=Node,RBAC`, `--enable-admission-plugins`. Each is a "knob" on control-plane behavior. Pay special attention to `--authorization-mode` and the admission plugins — we'll get to them in Lab 4.

## 1.2 kube-apiserver — the gateway

The apiserver is the most central component. Its jobs:

1. **Expose the API** — RESTful, with group/version/resource (e.g. `apps/v1/deployments`).
2. **Authentication** — who are you? (certificates, tokens, OIDC…).
3. **Authorization** — are you allowed? (mostly RBAC).
4. **Admission Control** — a chain of "gatekeepers" that can *mutate* or *reject* requests before they're persisted.
5. **Validation + persistence** — write to etcd.
6. **Watch semantics** — clients can "subscribe" to changes instead of polling.

### The request path

```
kubectl apply
   │
   ▼
[ Authentication ]  →  who are you?
   │
   ▼
[ Authorization (RBAC) ]  →  are you allowed to perform this action?
   │
   ▼
[ Mutating Admission ]  →  webhooks/plugins that modify the object (e.g. inject a sidecar)
   │
   ▼
[ Schema Validation ]  →  is the object valid?
   │
   ▼
[ Validating Admission ]  →  webhooks/plugins that approve/reject (e.g. policy)
   │
   ▼
[ etcd write ]  →  persisted. This is now the "desired state".
```

## Lab 2 — Talk to the API server directly (and watch watch)

`kubectl` is just an HTTP client. Let's bypass it and see what's underneath.

```bash
# The easy way: kubectl proxy opens an authenticated proxy to the API
$ kubectl proxy --port=8080 &

# Now plain old curl
$ curl -s http://localhost:8080/api/v1/namespaces/default/pods | head -30

# Discover all available API groups
$ curl -s http://localhost:8080/apis | python3 -m json.tool | grep '"name"'
```

Now the cool part — **watch**. This is the mechanism every controller and kubelet uses:

```bash
# Open one terminal with a watch — it stays open and "streams" events
$ curl -s "http://localhost:8080/api/v1/namespaces/default/pods?watch=true"

# In a second terminal, create a pod:
$ kubectl run nginx --image=nginx

# Back in the first terminal — you'll see a JSON stream of ADDED / MODIFIED
# in real time as the pod goes Pending → ContainerCreating → Running
```

**What you learned:** the API is not a static database — it's an **event stream**. Every control-plane component is essentially a client that "subscribes" (watch) to a resource type and reacts to changes. This is the heart of the reconciliation model.

## Lab 3 — Look directly inside etcd

etcd is a distributed key-value store (based on the Raft consensus algorithm). In Kubernetes, every object is stored under a key that is essentially its API path.

```bash
$ docker exec -it lab-control-plane bash

# etcdctl is already installed inside the node. It needs the apiserver's certs to connect
root@lab-control-plane:/# export ETCDCTL_API=3
root@lab-control-plane:/# alias e='etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

# All Kubernetes keys start with /registry
root@lab-control-plane:/# e get /registry --prefix --keys-only | head -40

# For example, the pod we created:
root@lab-control-plane:/# e get /registry/pods/default/nginx --print-value-only | head

# The value is stored as protobuf (not JSON) for efficiency — you'll see "weird"
# characters, but you'll recognize field names. It's exactly the same object
# you saw with kubectl get pod -o yaml.
```

**An important operational-maturity point:** if etcd is lost and there's no backup — the cluster is lost. Backing up etcd (`etcdctl snapshot save`) is the single most critical thing in control-plane maintenance. Try:

```bash
root@lab-control-plane:/# e snapshot save /tmp/backup.db
root@lab-control-plane:/# e snapshot status /tmp/backup.db --write-out=table
```

## 1.3 kube-scheduler — how a Pod finds a home

When you create a Pod, its `spec.nodeName` field is initially **empty**. The Pod exists in etcd but isn't running anywhere. The scheduler:

1. **Watches** pods with an empty `nodeName` (pending).
2. For each such pod runs two phases:
   - **Filtering (Predicates)** — eliminates nodes that don't fit: not enough CPU/memory, taints the pod doesn't tolerate, nodeSelector/affinity that isn't satisfied, occupied ports.
   - **Scoring (Priorities)** — scores the remaining nodes: spreading load, resource locality, least/most allocated, etc.
3. Picks the highest-scoring node and performs a **bind** — i.e. writes `nodeName` through the apiserver.

Note: the scheduler **does not run** the pod. It only *decides* and *writes* the decision. The kubelet on the relevant node is the one that sees (watch) a pod with its own `nodeName` and actually runs it.

## Lab 4 — Watch the scheduling decision + deliberately break it

```bash
# Create a pod and see the events — including the Scheduled line from the scheduler
$ kubectl run demo --image=nginx
$ kubectl describe pod demo | grep -A10 Events
# You'll see: "Successfully assigned default/demo to lab-worker"  ← the scheduler's decision

# Now let's make a pod stuck in Pending on purpose: request more CPU than any node has
$ kubectl run greedy --image=nginx --overrides='
{"spec":{"containers":[{"name":"greedy","image":"nginx",
 "resources":{"requests":{"cpu":"100"}}}]}}'

$ kubectl get pod greedy          # stays Pending
$ kubectl describe pod greedy | grep -A10 Events
# You'll see: "0/3 nodes are available: 3 Insufficient cpu"  ← the scheduler explaining why filtering failed

$ kubectl delete pod greedy demo
```

**Extension exercise:** try a `taint` on a node and watch the scheduler avoid it:
```bash
$ kubectl taint nodes lab-worker key=value:NoSchedule
$ kubectl run t --image=nginx   # see which node it lands on (not lab-worker)
$ kubectl taint nodes lab-worker key=value:NoSchedule-   # removes the taint
```

## 1.4 kube-controller-manager — the heart of reconciliation

This is a single process that contains **many controllers** that run in loops. Each controller is responsible for a resource type:

- **Deployment controller** — ensures the right ReplicaSets exist for each Deployment.
- **ReplicaSet controller** — ensures the actual number of pods = the desired `replicas`.
- **Node controller** — detects nodes that have gone down and marks them.
- **Job / CronJob, Endpoints, ServiceAccount, Namespace** controllers — and dozens more.

### Anatomy of a controller (this is the pattern that recurs in every operator)

```
for {
    desired  := read from API what the desired state is (watch/informer)
    observed := read from API what is actually happening
    diff     := desired - observed
    if diff != 0 {
        take an action to shrink the gap (create/delete/update via apiserver)
    }
    // loops again and again — level-triggered, not edge-triggered
}
```

The difference between **level-triggered** and edge-triggered is critical: even if the controller missed an event (crashed and came back), on the next pass it compares states and fixes things. The system "self-heals". This is why Kubernetes is resilient to failures.

## Lab 5 — Watch reconciliation heal itself

```bash
$ kubectl create deployment web --image=nginx --replicas=3
$ kubectl get pods -l app=web        # 3 pods

# Now "sabotage" — delete a pod manually. The controller should bring it back immediately:
$ kubectl delete pod $(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')

# Run immediately:
$ kubectl get pods -l app=web -w
# You'll see a new pod created within seconds — the ReplicaSet controller detected 2≠3 and fixed it

# Another experiment: change the replica count and see who reacts
$ kubectl scale deployment web --replicas=5
$ kubectl get rs -l app=web       # the ReplicaSet was updated to 5

$ kubectl delete deployment web
```

**What you learned:** nobody "created the new pod" directly. You declared `replicas: 3`, and the loop converged there. This is the fundamental difference between Kubernetes and a plain container runner.

## Lab 6 — Build a minimal "controller" yourself (bash)

To internalize the pattern, we'll write a tiny reconciler that ensures a certain ConfigMap always exists. This is exactly what an operator does, just without a framework.

```bash
$ cat > tiny-controller.sh <<'EOF'
#!/usr/bin/env bash
# Reconciler: ensures a configmap named "must-exist" exists with the right value
DESIRED_KEY="owner"; DESIRED_VAL="lab"
while true; do
  current=$(kubectl get configmap must-exist -o jsonpath="{.data.$DESIRED_KEY}" 2>/dev/null)
  if [ "$current" != "$DESIRED_VAL" ]; then
    echo "$(date +%T)  drift detected (got '$current') → reconciling"
    kubectl delete configmap must-exist --ignore-not-found >/dev/null
    kubectl create configmap must-exist --from-literal=$DESIRED_KEY=$DESIRED_VAL >/dev/null
  fi
  sleep 3
done
EOF
$ chmod +x tiny-controller.sh
$ ./tiny-controller.sh &

# Now "sabotage" from another terminal:
$ kubectl delete configmap must-exist          # the controller brings it back
$ kubectl patch configmap must-exist --type merge -p '{"data":{"owner":"hacker"}}'  # the controller fixes it

$ kill %1   # stop the controller
```

This is exactly the principle behind every **Operator** and **Custom Resource Definition (CRD)**: you define your own new resource type + write a controller that runs this loop. The Kubernetes control plane is essentially an *extensibility framework* built around this pattern.

## 1.5 Mental summary for Part A

- **Everything goes through the apiserver.** Only it touches etcd.
- **etcd = source of truth.** Back it up.
- **Everything is declarative + level-triggered.** You declare a desired state; loops converge to it.
- **The scheduler decides, the kubelet runs, controllers fix.** Clean separation of concerns.
- **Watch, not polling.** The API is an event stream, and that's what enables responsiveness and scalability.

---

# Part B — eBPF

## 2.1 What eBPF is and why it changes everything

**eBPF** (extended Berkeley Packet Filter) lets you run small, safe programs **inside the Linux kernel**, in response to events — without writing a kernel module and without recompiling the kernel.

Why is this revolutionary? Before eBPF, if you wanted to intervene in what happens in the kernel (networking, security, tracing) you had two bad options: (a) write a kernel module — dangerous, a crash in it takes down the machine; (b) copy the data to user space and process it there — slow because of context switches. eBPF offers a third way: code that runs **at kernel speed** but **with sandbox safety**.

The idea: you write a small program, attach it to a **hook** in the kernel (entry to a syscall, receipt of a packet, etc.), and every time the event fires — your program runs.

### The three components that make it safe and fast

1. **The Verifier** — before an eBPF program is loaded, a static analyzer in the kernel verifies it is **safe**: no infinite loops (loops used to be forbidden entirely; today bounded loops are allowed), no out-of-bounds memory access, every execution path terminates. If the program doesn't pass — it simply won't load. This is the guarantee that buggy code won't crash the kernel.
2. **JIT compilation** — the verified program is translated from eBPF bytecode into the CPU's native machine code. That's why it runs *fast* — almost like native kernel code.
3. **Maps** — data structures (hash tables, arrays, ring buffers…) that survive across program invocations **and** are shared between the kernel and user space. This is how an eBPF program "talks" to your user-space program and keeps state.

```
   C code  ──(clang/LLVM)──►  eBPF bytecode
                                  │
                                  ▼
                          ┌───────────────┐
                          │   VERIFIER    │  ← rejects unsafe code
                          └───────┬───────┘
                                  ▼
                          ┌───────────────┐
                          │      JIT      │  → native machine code
                          └───────┬───────┘
                                  ▼
              ┌───────────── HOOK in the kernel ──────────┐
              │ syscall / packet / kprobe / tracepoint     │
              └───────────────────┬────────────────────────┘
                                  │  read/write
                                  ▼
                          ┌───────────────┐
                          │     MAPS      │ ◄──► user-space program
                          └───────────────┘
```

## 2.2 Program types and hooks — where you can "attach"

The power of eBPF is in the variety of points where you can attach a program. The main categories:

**Tracing & Observability** (observe, without modifying):
- **kprobe / kretprobe** — attach to any kernel function, on entry or return. Very flexible but tied to the kernel version.
- **tracepoint** — stable tracing points the kernel defines officially (preferred over kprobe when available).
- **uprobe** — like kprobe but on user-space functions (e.g. a function inside an application).
- **perf events** — counter-based sampling (CPU profiling).

**Networking** (observe *and modify* traffic):
- **XDP (eXpress Data Path)** — the earliest possible point: runs at the NIC driver *before* the kernel even built an sk_buff. Extremely fast — used for DDoS mitigation and load balancing (this is what Cilium/Katran exploit).
- **tc (traffic control)** — slightly later in the stack, but with more context. Used for network policies.
- **socket filters / cgroup hooks** — at the socket or cgroup level (the basis for a sidecar-less service mesh).

**Security**:
- **LSM (Linux Security Modules) hooks** — enforcing security policy (the basis for Tetragon, and Falco in some modes).
- **seccomp-bpf** — filtering syscalls (the classic one, predating modern eBPF).

## Lab 7 — Prepare the environment + bpftrace hello world

`bpftrace` is the fastest way to get started: a high-level scripting language over eBPF, awk-style. Perfect for learning.

```bash
# Requires a real Linux kernel (not a container alone). On a VM/physical machine:
$ sudo apt-get update && sudo apt-get install -y bpftrace bpfcc-tools linux-headers-$(uname -r)

# Sanity check — prints a version
$ bpftrace --version

# "Hello world": print a line every time any process calls execve (runs a new program)
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s called execve\n", comm); }'
# Open another terminal and run some commands — you'll see them appear here in real time. Ctrl-C to stop.
```

> Important note: eBPF needs kernel access. If you're on macOS/Windows or inside a managed container — run inside a Linux VM (Multipass/Lima/VirtualBox) or WSL2 with a recent kernel. In kind/Docker alone you usually *cannot* load eBPF programs because you don't have full kernel privileges.

## Lab 8 — bpftrace one-liners that teach a lot

```bash
# 1. Count syscalls by type, across all processes, for a few seconds (Ctrl-C for the summary)
$ sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[probe] = count(); }'

# 2. Who opens files and which ones? (tracking openat)
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%-16s %s\n", comm, str(args->filename)); }'

# 3. A histogram of read() durations — how long each read takes (latency distribution)
$ sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_read  { @start[tid] = nsecs; }
  tracepoint:syscalls:sys_exit_read  /@start[tid]/ {
     @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'
# On Ctrl-C you get a nice log-scale histogram — a real profiling tool

# 4. Who creates new TCP connections?
$ sudo bpftrace -e 'kprobe:tcp_connect { printf("%s -> connecting\n", comm); }'
```

**What's happening conceptually:** `@name` is a **map** (exactly the structure from 2.1). `count()`, `hist()`, `@start[tid]` — all use maps to accumulate state between events. `comm`, `tid`, `args->…` are the context variables the kernel provides at the hook. You write logic that runs *inside the kernel* on every event, accumulating results in a map that's read out at the end.

## Lab 9 — A real bpftrace script (file)

We'll write a "mini-monitor" that counts how many bytes each process sends over the network.

```bash
$ cat > netbytes.bt <<'EOF'
#!/usr/bin/env bpftrace
// counts bytes sent (write on a socket) by process name

BEGIN { printf("Tracing network sends... Ctrl-C to end.\n"); }

tracepoint:syscalls:sys_enter_sendto,
tracepoint:syscalls:sys_enter_write
{
    @bytes[comm] = sum(args->count);   // map: process name → sum of bytes
}

END { printf("\n=== bytes sent per process ===\n"); print(@bytes); }
EOF

$ sudo bpftrace netbytes.bt
# Generate network load in another terminal (curl, apt update...) then Ctrl-C to see the table
```

## 2.3 The layers: bpftrace ↔ BCC ↔ libbpf ↔ Go

There are three "maturity layers" for writing eBPF, from easy to professional:

| Tool | Language | When to use |
|------|----------|-------------|
| **bpftrace** | awk-style DSL | Quick, ad-hoc exploration, one-liners. Fastest to learn. |
| **BCC** | Python + embedded C | More complex scripts, many ready-made tools (`/usr/share/bcc/tools`). |
| **libbpf + CO-RE** | Pure C, compiled ahead of time | Production. "Compile Once – Run Everywhere" — one binary that runs across kernel versions. |
| **cilium/ebpf** | Go | Production in Go, no runtime dependency on libc/clang. What Cilium itself uses. |

## Lab 10 — Explore the ready-made BCC tools

BCC ships with dozens of ready-made tools that are themselves excellent learning examples:

```bash
$ ls /usr/share/bcc/tools/    # or /usr/sbin/*-bpfcc on some installs

# Recommended examples (run, then Ctrl-C):
$ sudo /usr/share/bcc/tools/execsnoop     # every new process created, with its args
$ sudo /usr/share/bcc/tools/opensnoop     # every file opened
$ sudo /usr/share/bcc/tools/tcpconnect    # every outgoing TCP connection
$ sudo /usr/share/bcc/tools/biolatency    # disk latency histogram
$ sudo /usr/share/bcc/tools/profile       # eBPF-based CPU profiler
```

**Exercise:** open the source of `execsnoop` (`less $(which execsnoop-bpfcc)` or under tools). You'll see a C part (the program that runs in the kernel, attaches a kprobe to `execve` and writes to a map) and a Python part (loads it, reads from the map, prints). This is exactly the architecture from 2.1 in real code.

## Lab 11 — A minimal libbpf program (C, optional)

This is the "professional" layer. If you want to experiment, here's a small XDP program that counts packets. (Optional — requires clang + libbpf-dev.)

```bash
$ sudo apt-get install -y clang llvm libbpf-dev linux-tools-common

# The kernel program: counts every packet coming in via XDP
$ cat > xdp_count.bpf.c <<'EOF'
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, __u64);
} pkt_count SEC(".maps");

SEC("xdp")
int count_packets(struct xdp_md *ctx) {
    __u32 key = 0;
    __u64 *val = bpf_map_lookup_elem(&pkt_count, &key);
    if (val) __sync_fetch_and_add(val, 1);
    return XDP_PASS;   // pass the packet along (don't drop)
}
char _license[] SEC("license") = "GPL";
EOF

# Compile to an eBPF object
$ clang -O2 -g -target bpf -c xdp_count.bpf.c -o xdp_count.bpf.o

# Load and attach to an interface (e.g. eth0 or lo); requires bpftool
$ sudo bpftool net attach xdp obj xdp_count.bpf.o sec xdp dev lo
$ sudo bpftool map dump name pkt_count      # watch the counter climb
$ sudo bpftool net detach xdp dev lo         # detach
```

Note the `return XDP_PASS` — an XDP program *decides what to do with each packet*: `XDP_PASS` (continue normally), `XDP_DROP` (drop it — this is how you build a fast firewall), `XDP_TX` (send it back to the sender), `XDP_REDIRECT` (forward to another NIC — the basis for load balancing). This is exactly what makes eBPF such a powerful networking tool.

## 2.4 Mental summary for Part B

- eBPF = run small, safe code **inside the kernel**, in response to events, without a kernel module.
- **Verifier + JIT + Maps** = safety + speed + state/communication with user space.
- **Hooks** determine *when* the code runs: tracing (kprobe/tracepoint/uprobe), networking (XDP/tc), security (LSM).
- Learning path: start with **bpftrace** → ready-made BCC tools → libbpf/Go for production.

---

# Part C — The connection: eBPF inside Kubernetes

Now that both topics are clear, let's see where they meet. The control plane decides *what* should run; eBPF optimizes *how* those workloads' networking, security, and observability work in the data plane.

## 3.1 The problem with classic kube-proxy

Recall from Part A that a **Service** in Kubernetes provides a stable virtual IP in front of a set of changing pods. The one that implements this on each node is **kube-proxy**. Traditionally it does so with **iptables** (or ipvs): for each Service it writes a chain of iptables rules mapping the ClusterIP to one of the pods.

The problem: iptables is a **linear list**. With thousands of services, every packet has to traverse thousands of rules in order — performance degrades linearly with cluster size, and updating the rules becomes slow. This is a well-known bottleneck in large clusters.

## 3.2 How eBPF solves it (Cilium)

**Cilium** is a CNI (Container Network Interface) that replaces kube-proxy with eBPF. Instead of a linear iptables list, it uses **eBPF maps** (hash tables — O(1) lookup) and programs attached at XDP/tc:

- **kube-proxy replacement** — instead of thousands of iptables rules, a single hash table in a map. Service→Pod translation is a constant-time lookup, independent of cluster size.
- **Network Policies** — enforcement in eBPF at the tc/socket level, including L7-level policy (HTTP paths, not just IP/port).
- **Observability (Hubble)** — because every packet already passes through an eBPF program, you can export metrics and accurate network flows "for free" without sidecars.
- **Sidecar-less service mesh** — socket-level hooks enable mTLS/routing without injecting a proxy into every pod.

Nice tie-back to Part A: when you create a `Service` via the apiserver, the Cilium agent (which is essentially a **controller** that watches Services and Endpoints — exactly the reconciliation pattern from Lab 6!) translates the desired state into eBPF maps on each node. Same pattern again: control plane declares, agent reconciles, eBPF enforces in the kernel.

## Lab 12 — A cluster with Cilium and eBPF observability

kind supports disabling kube-proxy so Cilium can fully replace it with eBPF:

```bash
# A cluster without kube-proxy
$ cat > kind-cilium.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true      # don't install the default CNI
  kubeProxyMode: none          # no kube-proxy — Cilium takes its place
nodes: [ {role: control-plane}, {role: worker} ]
EOF
$ kind create cluster --name cil --config kind-cilium.yaml

# Install the Cilium CLI + Cilium
$ CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
$ curl -L --fail -o cilium.tar.gz \
   https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
$ sudo tar xzf cilium.tar.gz -C /usr/local/bin

$ cilium install --set kubeProxyReplacement=true
$ cilium status --wait          # wait for green

# Enable Hubble — eBPF-based network observability
$ cilium hubble enable
$ cilium hubble port-forward &
$ hubble observe --follow       # see live network flows in the cluster, collected via eBPF
```

**What you saw here:** no kube-proxy and no iptables for service routing — it's all eBPF. And `hubble observe` shows every network flow in the cluster, because every packet already passes through an eBPF program that can report on it. This is the full connection between the two topics.

## Lab 13 (challenge) — A Network Policy enforced in eBPF

```bash
# Deploy two pods and verify they can talk
$ kubectl create deployment web --image=nginx
$ kubectl expose deployment web --port=80
$ kubectl run client --image=nicolaka/netshoot -it --rm -- curl -s web    # works

# Now add a policy that blocks everything, and show the connection is blocked — enforced in eBPF, not iptables
$ kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all }
spec:
  podSelector: { matchLabels: { app: web } }
  policyTypes: [ Ingress ]
EOF
$ kubectl run client --image=nicolaka/netshoot -it --rm -- curl -m3 -s web  # fails (timeout)

# In hubble you'll see the drops marked:
$ hubble observe --verdict DROPPED --follow
```

---

# Appendices

## How to set up an environment from scratch (if you don't have one)

**The easy option — a Linux VM, cloud or local:**
- Multipass: `multipass launch --name lab --cpus 4 --memory 8G --disk 40G` then `multipass shell lab`.
- Or any VM with Ubuntu 22.04+ (important: kernel 5.15+ for most modern eBPF features).

**What to install all at once:**
```bash
$ sudo apt-get update && sudo apt-get install -y \
    docker.io bpftrace bpfcc-tools linux-headers-$(uname -r) \
    clang llvm libbpf-dev
$ sudo usermod -aG docker $USER && newgrp docker
# then kind + kubectl per Lab 0
```

> Important: **eBPF requires a real kernel with full privileges.** Labs 7–13 won't work inside a plain container or Docker Desktop on Mac/Windows — run them in a Linux VM. The control-plane labs (0–6) work anywhere Docker runs.

## A roadmap for going deeper (after this guide)

For the **control plane**: writing a real Operator with Kubebuilder or the Operator SDK (Go) — the natural continuation of Lab 6. After that: admission webhooks (mutating/validating), understanding RBAC in depth, and etcd internals (Raft, compaction, defrag).

For **eBPF**: Brendan Gregg's book "BPF Performance Tools" is the bible for observability. For networking/security: the Cilium and Tetragon docs. For writing: the `cilium/ebpf` (Go) library with its examples, and CO-RE for understanding portability across kernel versions.

**The most interesting connection to pursue next:** Tetragon (from the Cilium project) — eBPF-based runtime security enforcement in Kubernetes, which shows how a declarative control plane (policy CRDs) meets eBPF hooks at the kernel level. This is exactly the intersection of the two topics you learned.

## Quick-reference command table

| Goal | Command |
|------|---------|
| See control-plane components | `kubectl get pods -n kube-system` |
| Enter a kind node | `docker exec -it <cluster>-control-plane bash` |
| Watch the API | `curl ".../pods?watch=true"` via `kubectl proxy` |
| Why a pod is stuck | `kubectl describe pod X` → Events section |
| Back up etcd | `etcdctl snapshot save backup.db` |
| eBPF hello world | `bpftrace -e 'tracepoint:syscalls:sys_enter_execve {...}'` |
| Ready-made tools | `ls /usr/share/bcc/tools/` |
| Cilium status | `cilium status` · `hubble observe --follow` |
