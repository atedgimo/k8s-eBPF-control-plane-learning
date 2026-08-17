# k8s-eBPF-control-plane-learning

Hands-on course: Kubernetes control plane internals + eBPF, tied together through Cilium.
Completed Aug 2026 on an Apple Silicon Mac (OrbStack Docker + Multipass Ubuntu VM).

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

## The course in one sentence

The control plane declares desired state in etcd, controllers watch and reconcile —
and in the modern stack, reconciliation bottoms out in eBPF programs and maps inside the kernel.
