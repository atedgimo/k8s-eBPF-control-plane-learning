# eBPF — the picture in my head

> Companion reference: the lifecycle of an eBPF program, and a map of the hooks —
> where in the kernel each one fires, and which labs used them.

## 1. The lifecycle of every eBPF program

Same pipeline whether it's a bpftrace one-liner or Cilium — only the tooling differs.

```
  my C / bpftrace code
        │
        │  clang -target bpf        (bpftrace does this invisibly; Lab 11 does it by hand)
        ▼
  eBPF bytecode (.o)
        │
        │  load (bpftool / bpf() syscall)
        ▼
 ┌───────────────────┐
 │     VERIFIER      │  static proof of safety: bounded loops, no wild pointers,
 │                   │  every path terminates. Fails → program never loads.
 └─────────┬─────────┘  This is why eBPF can't crash the kernel.
           ▼
 ┌───────────────────┐
 │       JIT         │  bytecode → native machine code. This is why it's fast.
 └─────────┬─────────┘
           ▼
   attached to a HOOK  ──── event fires ────► my program runs, in kernel context
           │
           │ reads/writes
           ▼
 ┌───────────────────┐
 │       MAPS        │ ◄──── userspace reads results (bpftool map dump / bpftrace @ / Cilium agent)
 └───────────────────┘
```

**The triangle:** hook = *when* my code runs · program = *what* it does · map = *where state lives*.

## 2. Where the hooks live

Two journeys pass through the kernel: a **syscall going down** (app → kernel) and a
**packet coming up** (NIC → app). Hooks are parked along both roads.

```
              USER SPACE
   ┌────────────────────────────────┐
   │  my app        nginx, bash ... │
   │   ▲ uprobe — attach to any     │
   │   │          user-space func   │
   └───┼──────────────┬─────────────┘
       │              │ syscall (execve, read, sendto…)
═══════╪══════════════╪═══ syscall boundary ═══════════════════════
       │              ▼
       │   ● tracepoint:syscalls:*     ← Labs 7–8 (execve, openat, read)
       │   ● seccomp-bpf (filter/deny syscalls — the oldest hook)
       │
   K   │   ● kprobe / kretprobe — any kernel function, entry/return
   E   │        e.g. kprobe:tcp_connect ← Lab 8
   R   │   ● tracepoint (non-syscall) — stable, kernel-defined events
   N   │   ● LSM hooks — security decision points ("may X do Y?")
   E   │        → Tetragon enforces here
   L   │
       │        NETWORK STACK (packet path, bottom → top)
       │   ┌──────────────────────────────────────┐
       │   │ ● socket / cgroup hooks              │  ← per-socket, per-cgroup;
       │   │      sidecar-less service mesh       │     sees "which pod"
       │   │ ● tc (traffic control)               │  ← full packet + metadata;
       │   │      Cilium network policy ← Lab 13  │     can modify/redirect
       │   │ ● XDP — in the NIC driver, BEFORE    │  ← earliest + fastest;
       │   │      the kernel builds the packet    │     Lab 11, DDoS/LB
       │   │      struct  ← Lab 11                │
       │   └──────────────▲───────────────────────┘
       └──────────────────┤
                      ┌───┴───┐
                      │  NIC  │
                      └───────┘
```

## 3. The hooks, one table

| Hook | Fires when | Stable API? | Can modify? | Typical use | Seen in |
|------|-----------|-------------|-------------|-------------|---------|
| **tracepoint** | kernel hits a defined trace point (syscalls, sched, …) | ✅ yes — prefer it | no (observe) | tracing, counting, latency | Labs 7–8 |
| **kprobe / kretprobe** | entry/return of *any* kernel function | ❌ tied to kernel version | no (observe) | tracing when no tracepoint exists | Lab 8 (`tcp_connect`) |
| **uprobe** | entry/return of a user-space function | per-binary | no | app-level tracing (e.g. SSL_read) | — |
| **perf event** | timer/counter sample (e.g. 99 Hz) | ✅ | no | CPU profiling, flamegraphs | `profile-bpfcc` |
| **XDP** | packet arrives at NIC driver | ✅ | ✅ verdict: PASS / DROP / TX / REDIRECT | firewall, DDoS, load balancing | Lab 11 |
| **tc** | packet in traffic-control layer (ingress+egress) | ✅ | ✅ | network policy, encap — Cilium's workhorse | Labs 12–13 |
| **socket / cgroup** | socket ops (connect, send…) per cgroup | ✅ | ✅ | pod-level policy, service mesh without sidecars | Cilium |
| **LSM** | security decision points | ✅ | ✅ allow/deny | runtime security enforcement | Tetragon |
| **seccomp-bpf** | each syscall of a process | ✅ | ✅ allow/deny | syscall sandboxing (Docker/K8s profiles) | — |

## 4. How to choose (rules of thumb)

- **Observing kernel behavior?** tracepoint if one exists, kprobe otherwise (accept version fragility).
- **Observing an application's internals?** uprobe.
- **Touching packets as early/fast as possible, drop/redirect?** XDP.
- **Touching packets but need more context (direction, netns, marks)?** tc.
- **Policy per pod/container rather than per interface?** socket/cgroup hooks.
- **Allow/deny security-relevant actions?** LSM (modern) or seccomp (syscalls only).

## 5. Verdicts cheat-line (XDP)

`XDP_PASS` continue up the stack · `XDP_DROP` discard (firewall) ·
`XDP_TX` bounce back out same NIC · `XDP_REDIRECT` send to another NIC/CPU (load balancer).

## 6. Where Part C plugs in

Cilium = a **controller** (Part A pattern: watches Services/Endpoints/Policies via the apiserver)
whose *actuator* is eBPF: it compiles policy into programs at **tc/XDP/socket** hooks and
state into **maps** (the O(1) replacement for kube-proxy's iptables chains).
Control plane declares → agent reconciles → kernel enforces.
