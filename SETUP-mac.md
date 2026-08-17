# SETUP (macOS) — Environment for the k8s + eBPF Lab

> This file gets you from a clean Mac to a working lab. It has two stages:
> **Stage 1** runs natively on macOS and covers all of Part A (Control Plane, Labs 0–6).
> **Stage 2** sets up a Linux VM required for Part B/C (eBPF, Labs 7–13).
>
> Don't do everything up front. Do Stage 1, complete Part A, and only set up the VM when you reach eBPF.

---

## Know your chip first

Run this once — it determines which binaries you download:

```bash
$ uname -m
# arm64  → Apple Silicon (M1/M2/M3/M4)
# x86_64 → Intel Mac
```

- **Apple Silicon (`arm64`)**: use the `arm64` builds. kind, kubectl, cilium, bpftrace all support arm64. Occasionally a container image or tool exists only for x86 — if something fails with an "exec format error" or "no matching manifest", that's the tell; ask Claude Code for the arm64 alternative.
- **Intel (`x86_64`)**: use the `amd64` builds everywhere.

Below, wherever a download URL contains `arm64` or `amd64`, pick the one matching your chip.

---

## Stage 1 — Native on macOS (Part A: Control Plane)

Everything in Part A runs on the Mac itself. No VM needed.

### 1. Install Homebrew (if you don't have it)

```bash
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. Install Docker, kind, and kubectl

The simplest path is Docker Desktop; Colima is a lighter alternative. Either works.

```bash
# Option A — Docker Desktop (GUI, easiest)
$ brew install --cask docker
# then launch Docker Desktop from Applications once, so the engine starts

# Option B — Colima (lightweight, CLI-only) — pick ONE option, not both
# $ brew install colima docker && colima start

# kind + kubectl (Homebrew picks the right arch automatically)
$ brew install kind kubectl

# sanity checks
$ docker ps          # should not error
$ kind version
$ kubectl version --client
```

### 3. Create the cluster (Lab 0)

```bash
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
$ kubectl get pods -n kube-system      # you should see the control-plane components
```

You're now ready for Labs 0–6. Go to the main guide.

> Note: on a Mac, `docker exec -it lab-control-plane bash` (used in Labs 1 and 3 to go inside the node) works the same — the kind nodes are Docker containers running on the Mac's Docker engine.

---

## Stage 2 — Linux VM (Part B/C: eBPF)

eBPF runs *inside* the Linux kernel. macOS has no Linux kernel, and Docker Desktop's internal VM doesn't grant the privileges eBPF needs. So Labs 7–13 require a real Linux VM. We'll use **Multipass** (simplest on Mac).

### 1. Install Multipass and launch a VM

```bash
$ brew install --cask multipass

# Launch an Ubuntu VM (Multipass auto-selects arm64 on Apple Silicon, amd64 on Intel)
$ multipass launch 22.04 --name ebpf --cpus 4 --memory 8G --disk 40G

# Enter the VM
$ multipass shell ebpf
```

Everything from here runs **inside the VM** (your prompt will change to `ubuntu@ebpf`).

### 2. Install the eBPF toolchain (inside the VM)

```bash
ubuntu@ebpf:~$ sudo apt-get update
ubuntu@ebpf:~$ sudo apt-get install -y \
    bpftrace bpfcc-tools linux-headers-$(uname -r) \
    clang llvm libbpf-dev linux-tools-common

# verify
ubuntu@ebpf:~$ bpftrace --version

# hello world — should stream lines as you run commands in another 'multipass shell ebpf'
ubuntu@ebpf:~$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s\n", comm); }'
```

If `bpftrace` errors on load, it's almost always a kernel-headers mismatch — see the rescuer prompt below.

### 3. (For Part C) Docker + kind inside the VM

To run the Cilium labs (12–13) with real eBPF datapath, run kind *inside the Linux VM*, not on the Mac:

```bash
ubuntu@ebpf:~$ sudo apt-get install -y docker.io
ubuntu@ebpf:~$ sudo usermod -aG docker $USER && newgrp docker
# install kind + kubectl inside the VM (Linux binaries):
ubuntu@ebpf:~$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-$(dpkg --print-architecture)
ubuntu@ebpf:~$ chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
ubuntu@ebpf:~$ curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/$(dpkg --print-architecture)/kubectl"
ubuntu@ebpf:~$ chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

Then follow Lab 12 inside the VM.

### Useful Multipass commands (run on the Mac)

```bash
$ multipass list                    # see your VMs and their IPs
$ multipass shell ebpf              # enter the VM
$ multipass stop ebpf               # pause it (frees Mac resources)
$ multipass start ebpf              # resume
$ multipass delete ebpf && multipass purge   # remove entirely
$ multipass mount . ebpf:/home/ubuntu/repo    # share your repo folder into the VM
```

> That last `mount` command is handy: it makes your GitHub repo (with both guide files) visible inside the VM, so you can read the labs there too.

---

## Working with Claude Code across the two stages

Claude Code runs on your Mac. For Stage 1 it can run and debug commands directly. For Stage 2, remember it's driving the Mac shell — when you're *inside* `multipass shell ebpf`, you're in a separate machine. Two good patterns:

- Keep Claude Code on the Mac and paste VM output back to it for diagnosis.
- Or install Claude Code inside the VM as well, for the eBPF portion.

**Rescuer prompt if eBPF setup breaks (paste into Claude Code):**
> "I'm on an [Apple Silicon / Intel] Mac using a Multipass Ubuntu VM. Inside the VM, `bpftrace` fails with: [paste the full error]. This is usually a kernel-headers or permissions issue. Diagnose it — check the running kernel vs the installed headers — and fix it. Explain what was mismatched."

**Prompt to have Claude Code do the whole VM setup for you:**
> "I'm on an [Apple Silicon / Intel] Mac. Set up a Multipass Ubuntu VM for the eBPF labs: install Multipass via brew, launch a 4-CPU/8GB VM named `ebpf`, then inside it install bpftrace, bpfcc-tools, matching kernel headers, clang, and libbpf-dev. Verify with a hello-world trace. Walk me through each step and explain what the VM gives me that macOS can't."

---

## TL;DR checklist

- [ ] `uname -m` → know your chip (arm64 vs x86_64)
- [ ] Stage 1: `brew install --cask docker` + `brew install kind kubectl` → do Labs 0–6
- [ ] Stage 2 (only when you reach eBPF): `brew install --cask multipass` → launch VM → install eBPF toolchain → do Labs 7–13 inside the VM
- [ ] Part A runs on the Mac; Part B/C runs inside the Linux VM. Don't mix them up.
