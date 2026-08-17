# k8s-eBPF-control-plane-learning

Hands-on course: Kubernetes control plane internals + eBPF, tied together through Cilium.
Completed Aug 2026 on an Apple Silicon Mac (OrbStack Docker + Multipass Ubuntu VM).

## Goal

Two topics that look unrelated, learned by building and breaking things, until they meet:

1. **How the Kubernetes control plane actually works** — not the diagram, the real thing:
   read etcd directly, talk to the apiserver with curl, watch the scheduler decide,
   sabotage a Deployment and watch reconciliation heal it, write a controller by hand.
2. **How eBPF runs code inside the Linux kernel** — from bpftrace one-liners to writing,
   compiling, and loading an XDP program in C, past the verifier, reading its maps.
3. **The meeting point (Cilium)** — a cluster with *no kube-proxy*, where a NetworkPolicy
   declared in the control plane becomes an eBPF verdict dropping packets in the kernel,
   observed live with Hubble.

**The course in one sentence:** the control plane declares desired state in etcd,
controllers watch and reconcile — and in the modern stack, reconciliation bottoms out
in eBPF programs and maps inside the kernel.

## Lab architecture

Two environments on one Mac, because Part A only needs Docker, but eBPF needs a real
Linux kernel with full privileges — which no container on macOS can provide:

```
┌───────────────────────── Mac (Apple Silicon, Darwin kernel) ─────────────────────────┐
│                                                                                      │
│  ┌───── OrbStack (Docker engine) ─────┐   ┌───── Multipass VM "ebpf" ─────────────┐  │
│  │                                    │   │   Ubuntu 22.04 · real Linux kernel    │  │
│  │  PART A (Labs 0–6)                 │   │                                       │  │
│  │  kind cluster "lab"                │   │  PART B (Labs 7–11)                   │  │
│  │  ┌──────────────────┐              │   │  eBPF toolchain on the VM kernel:     │  │
│  │  │ lab-control-plane│              │   │  bpftrace · BCC tools · clang ·       │  │
│  │  │  apiserver, etcd,│ lab-worker   │   │  bpftool (XDP packet counter)         │  │
│  │  │  scheduler,      │ lab-worker2  │   │                                       │  │
│  │  │  controller-mgr  │              │   │  PART C (Labs 12–13)                  │  │
│  │  └──────────────────┘              │   │  VM's own Docker                      │  │
│  │                                    │   │  └─ kind cluster "cil"                │  │
│  └────────────────────────────────────┘   │     no kube-proxy, no default CNI —   │  │
│                                           │     Cilium eBPF datapath + Hubble     │  │
│   kubectl contexts: kind-lab, orbstack    │     (kubectl context: kind-cil)       │  │
│                                           └───────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

Each kind "node" is a Docker container running its own kubelet + containerd, with the
control-plane components as static pods inside. In the VM, all `cil` nodes share the
VM's single kernel — which is exactly where Cilium loads its eBPF programs.

## The files

| File | What it is |
|------|-----------|
| [k8s-control-plane-and-ebpf-lab.md](k8s-control-plane-and-ebpf-lab.md) | The course itself — Parts A (control plane), B (eBPF), C (Cilium), Labs 0–13 |
| [SETUP-mac.md](SETUP-mac.md) | Environment setup: Stage 1 (Mac, Part A) and Stage 2 (Linux VM, Parts B/C) |
| [lab-with-claude-code-prompts.md](lab-with-claude-code-prompts.md) | Companion: how to learn this with Claude Code as rescuer/teacher, not executor |
| [CHEATSHEET.md](CHEATSHEET.md) | **My working notes** — every command as it actually ran on this setup, with all the guide-vs-reality fixes |
| [EBPF-HOOKS.md](EBPF-HOOKS.md) | eBPF reference: program lifecycle, the hook map, when to use which |
| [kind-config.yaml](kind-config.yaml) | Lab 0 cluster config (1 control-plane + 2 workers) |

## If commands differ

Trust **CHEATSHEET.md** over the guides — it records what actually worked here
(bpftool two-step load, arm64 downloads, etcdctl-via-exec, port 8001, hubble CLI install, …).
