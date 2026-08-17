# Lab Cheatsheet — my working commands

> Commands as they actually work on **my** setup (Apple Silicon Mac, OrbStack Docker, Multipass VM),
> including fixes the original guides don't have.

## Which machine am I on? (check the prompt first!)

| Prompt | Machine | Kernel |
|--------|---------|--------|
| `~ on ☁️ …` | Mac | Darwin — no eBPF, sudo asks password |
| `ubuntu@ebpf:~$` | Multipass VM | Linux — eBPF labs run HERE, sudo is passwordless |
| `root@lab-control-plane:/#` | kind node (container) | inside the k8s control-plane "machine" |

---

## Environments (run on the Mac)

```bash
# Multipass VM (eBPF labs)
multipass list                 # state + IP
multipass shell ebpf           # enter the VM
multipass stop ebpf            # free resources when done
multipass start ebpf

# kind cluster (control-plane labs)
kind get clusters
kubectl config use-context kind-lab    # labs cluster
kubectl config use-context orbstack    # back to my regular cluster
docker exec -it lab-control-plane bash # "SSH into" the control-plane node
```

---

## Pause / teardown / comeback (run on the Mac)

**Preferred wind-down** — keep what's expensive (VM toolchain), drop what's cheap (lab cluster):

```bash
kind delete cluster --name lab        # Part A cluster — nothing unique inside, 2 min to recreate
kubectl config use-context orbstack   # point kubectl back at my real cluster
multipass stop ebpf                   # VM preserved: toolchain + cil cluster + Cilium, zero CPU/RAM
```

**Comeback:**

```bash
multipass list && kubectl config current-context   # ALWAYS first: which machine, which cluster?
multipass start ebpf                               # eBPF world back, cil cluster included
# Part A again? recreate fresh (stale kind clusters wake up cranky after reboots):
kind create cluster --name lab --config kind-config.yaml
```

**Pause only** (short break, keep everything):

```bash
multipass stop ebpf
docker stop lab-control-plane lab-worker lab-worker2    # docker start ... revives
```

**Scorched earth** (done for good):

```bash
kind delete cluster --name lab
multipass delete ebpf && multipass purge   # kills everything inside: cil cluster, toolchain, all of it
kubectl config use-context orbstack
rm ~/tiny-controller.sh                    # stray from Lab 6
# optional: brew uninstall kind && brew uninstall --cask multipass
```

---

## Part A — Control plane (Mac, context kind-lab)

```bash
kubectl get pods -n kube-system                 # see the control plane
kubectl describe pod <pod> | grep -A10 Events   # why is a pod stuck/scheduled
kubectl get pods -l app=<x> -w                  # live watch

# API directly (NOT 8080 — OrbStack owns it)
kubectl proxy --port=8001 &
curl -s http://localhost:8001/api/v1/namespaces/default/pods | head -30
curl -s "http://localhost:8001/api/v1/namespaces/default/pods?watch=true"   # event stream

# Static pod manifests (inside the node)
ls /etc/kubernetes/manifests/
crictl ps                                       # containers one layer down

# etcd — etcdctl is NOT on the node anymore; exec into the etcd pod instead
alias e='kubectl exec -n kube-system etcd-lab-control-plane -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'
e get /registry --prefix --keys-only | head -40
e get /registry/pods/default/<pod> --print-value-only | head
e snapshot save /tmp/backup.db
# snapshot status moved to etcdutl in etcd 3.6:
kubectl exec -n kube-system etcd-lab-control-plane -- \
  etcdutl snapshot status /tmp/backup.db --write-out=table
```

---

## Part B — eBPF (inside the VM only)

```bash
# bpftrace patterns
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s\n", comm); }'   # hello world
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'          # count BY syscall
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'         # count BY process
# latency histogram (two-probe timestamp pattern)
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_read  { @start[tid] = nsecs; }
  tracepoint:syscalls:sys_exit_read  /@start[tid]/ {
     @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# BCC tools — Ubuntu names them *-bpfcc in /usr/sbin (not /usr/share/bcc/tools)
ls /usr/sbin/*-bpfcc
sudo execsnoop-bpfcc          # new processes
sudo tcpconnect-bpfcc         # outgoing TCP
sudo biolatency-bpfcc         # disk latency histogram
less $(which execsnoop-bpfcc) # read the source: Python + embedded C

# Lab 11 — build & load XDP (note the arm64 include flag!)
clang -O2 -g -target bpf -I/usr/include/aarch64-linux-gnu \
  -c xdp_count.bpf.c -o xdp_count.bpf.o
bpftool version               # if missing: sudo apt-get install -y linux-tools-$(uname -r)
# bpftool v5.15 can't load+attach in one shot ("expected 'id','tag','name' or 'pinned'"):
sudo bpftool prog load xdp_count.bpf.o /sys/fs/bpf/xdp_count   # ← verifier runs HERE
sudo bpftool prog show pinned /sys/fs/bpf/xdp_count            # confirm it's resident
sudo bpftool net attach xdp pinned /sys/fs/bpf/xdp_count dev lo
ping -c 5 localhost           # generate packets
sudo bpftool map dump name pkt_count
sudo bpftool net detach xdp dev lo
sudo rm /sys/fs/bpf/xdp_count # unpin (programs live while pinned)
```

---

## Part C — Cilium (inside the VM; separate cluster named `cil`)

```bash
# cluster WITHOUT kube-proxy and WITHOUT default CNI — Cilium takes both jobs
cat > kind-cilium.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
nodes: [ {role: control-plane}, {role: worker} ]
EOF
kind create cluster --name cil --config kind-cilium.yaml
kubectl get nodes            # NotReady until a CNI exists — expected!

# Cilium CLI — guide hardcodes amd64; this picks arm64 correctly:
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail -o cilium.tar.gz \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-$(dpkg --print-architecture).tar.gz
sudo tar xzf cilium.tar.gz -C /usr/local/bin

cilium install --set kubeProxyReplacement=true
cilium status --wait         # green = eBPF is now the datapath

# Hubble — network observability from the eBPF datapath
cilium hubble enable
# the hubble CLI is a SEPARATE binary the guide never installs:
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail -o hubble.tar.gz \
  https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-$(dpkg --print-architecture).tar.gz
sudo tar xzf hubble.tar.gz -C /usr/local/bin
cilium hubble port-forward &               # give it a second before observing (race!)
hubble observe --follow                    # every flow in the cluster
hubble observe --server 127.0.0.1:4245 --follow   # if "connection refused" on ::1
hubble observe --verdict DROPPED --follow  # only policy drops (Lab 13)

# look at the actual eBPF service map (kube-proxy's replacement)
kubectl -n kube-system exec ds/cilium -- cilium bpf lb list

# Lab 13 — deny-all policy, enforced in eBPF
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
kubectl run client --image=nicolaka/netshoot -it --rm -- curl -m3 -s web   # works
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all }
spec:
  podSelector: { matchLabels: { app: web } }
  policyTypes: [ Ingress ]
EOF
kubectl run client --image=nicolaka/netshoot -it --rm -- curl -m3 -s web   # times out
# watch the drops live: identity-based, two lines per SYN (verdict + drop)
hubble observe --verdict DROPPED --to-pod default/web --follow
kubectl delete networkpolicy deny-all      # undo
kubectl delete deployment web && kubectl delete service web   # full lab cleanup
```

---

## The toolbelt — who talks to whom

Every CLI is a thin client; the real machinery is always elsewhere (a daemon, a cluster, or the kernel).

**On the Mac:**

| CLI | Talks to |
|-----|----------|
| `multipass` | multipassd daemon → the `ebpf` VM |
| `docker` | OrbStack's engine (runs the `lab` kind nodes) |
| `kind` | Docker — only creates/destroys node containers, then idle |
| `kubectl` | the apiserver of the active context (`kind-lab` / `orbstack`) |

**In the VM:**

| CLI | Talks to |
|-----|----------|
| `bpftrace`, `*-bpfcc` | the kernel via the `bpf()` syscall (compile → load → read maps) |
| `clang` | nothing — just produces `.bpf.o` bytecode files |
| `bpftool` | the `bpf()` syscall directly: load / pin / attach / dump maps |
| `docker`, `kind` | the VM's own dockerd (separate world from OrbStack; runs the `cil` nodes) |
| `kubectl` | the `cil` cluster's apiserver (context `kind-cil`) |
| `cilium` | the **apiserver only** — installer + status reader; never touches eBPF itself |
| `hubble` | Hubble Relay (gRPC on :4245 via port-forward) |

**The two chains to memorize:**

```
config in:   cilium CLI → apiserver → agent pods → bpf() → kernel programs+maps
data out:    packet → eBPF prog (kernel) → agent → Relay → port-forward → hubble CLI
```

The datapath itself has no CLI — it's kernel state. Inspect it with `bpftool` or
`kubectl -n kube-system exec ds/cilium -- cilium bpf lb list` (from inside the agent pod).

---

## Quick rescues

```bash
lsof -nP -iTCP:<port> -sTCP:LISTEN   # who owns a port (Mac)
multipass list                        # VM alive? has IP? → retry shell once before debugging
kubectl config current-context       # am I talking to the cluster I think I am?
```

## The mental model (Part B)

Every eBPF system is the same triangle:
**hook** (when code runs) → **program** (logic in the kernel) → **map** (state, read from userspace).
bpftrace one-liners and Cilium differ only in engineering, not in model.

## The mental model (Part C)

Cilium = the Part A controller pattern with eBPF as its actuator:
**control plane declares** (Service/NetworkPolicy in etcd) → **agent reconciles** (watches apiserver)
→ **kernel enforces** (programs at tc/XDP/socket hooks, state in maps — O(1) vs iptables' linear chains).

Three kubectls now exist: Mac (`kind-lab` + `orbstack`) and one inside the VM (`kind-cil`). Check
`kubectl config current-context` before wondering why a cluster looks wrong.
