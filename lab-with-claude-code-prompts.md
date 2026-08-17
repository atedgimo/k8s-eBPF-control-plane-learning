# Companion Guide: Building the Lab with Claude Code

> This document accompanies "Hands-on Guide: Kubernetes Control Plane + eBPF".
> Guiding principle: **you type and analyze, Claude Code rescues and deepens.** Don't let it run whole labs for you — you'll lose the learning. Use it like a mentor sitting next to you.

## How to work with Claude Code in this session (read once)

**The setup that saves you headaches:** run Claude Code from a dedicated working directory (`mkdir k8s-ebpf-lab && cd $_ && claude`). That way all the YAML files and scripts you create live in one place.

**The three correct modes of use:**

1. **Rescuer (Debugging)** — when a command fails. Paste it the full error output and let it run diagnostic commands. This is the most powerful use.
2. **Teacher (Deepening)** — after a lab works, ask "why" and "what if". This is where understanding deepens.
3. **Accelerator (Boilerplate)** — for writing long YAML/C files where typing by hand teaches nothing (e.g. the libbpf program in Lab 11).

**What *not* to do:** don't ask "run all of Part A for me." Work lab by lab, type the key commands yourself, and let it accompany you. Rule of thumb: if typing *teaches* (kubectl, a bpftrace one-liner) — type it yourself. If typing is *tedious* (50 lines of C) — let it do it.

**Golden tip after every lab:** paste the output you got and ask: *"Explain exactly what happened here line by line, and what would change if ___"*. This turns technical output into insight.

**A note on permissions:** eBPF labs require `sudo` and kernel access. Claude Code runs commands with your privileges — pay attention to commands it proposes to run with sudo, and approve only what you understand.

---

# Part A — Control Plane

## Lab 0 — Spin up a cluster

**If you're stuck (rescuer prompt):**
> "I'm trying to run `kind create cluster` and it fails. Here's the full output: [paste]. Check whether Docker is running and configured correctly, diagnose the problem, and propose a fix. Don't install anything without first explaining to me what and why."

**To go deeper (teacher prompt):**
> "kind created 3 Docker containers as nodes. Run `docker ps` and explain what each container is, and how this differs from a real production cluster with separate machines."

## Lab 1 — See the control plane

**To go deeper:**
> "We're inside the control-plane node. Open `/etc/kubernetes/manifests/kube-apiserver.yaml` and walk me through the flags one by one — explain what each does and why that particular value was chosen. Focus on `--authorization-mode`, `--enable-admission-plugins`, and `--etcd-servers`."

**Guided challenge:**
> "What would happen if I changed a flag in this manifest file while the cluster is running? Explain the static-pod mechanism and what would actually happen. Don't change anything — just explain."

## Lab 2 — Talk to the API directly

**If you're stuck:**
> "`kubectl proxy` is running but curl to `localhost:8080` returns an error: [paste]. Diagnose it."

**To go deeper (this is the key watch moment):**
> "I ran curl with `?watch=true` and saw a JSON stream when I created a pod. Explain how this mechanism works underneath: what ResourceVersion is, how the apiserver knows where to resume from, and why all controllers use this instead of polling. Give a concrete example from the stream I saw."

## Lab 3 — etcd

**If you're stuck (common here — certificates):**
> "I'm trying to run etcdctl inside the node but I get a cert/connection error: [paste]. Help me build the correct etcdctl command with the right cacert/cert/key."

**To go deeper:**
> "I saw that the values in etcd are stored as protobuf and not JSON. Explain why Kubernetes chose protobuf, what it does for performance, and how the apiserver converts between that representation and the YAML I see in kubectl. Also explain what Raft is and why etcd needs an odd number of nodes."

## Lab 4 — scheduler

**To go deeper (after you saw 'Insufficient cpu'):**
> "I made a pod stuck in Pending with 'Insufficient cpu'. Now explain the two scheduling phases — filtering and scoring — using this situation as an example. What's the difference between requests and limits, and why does the scheduler look only at requests?"

**Guided challenge:**
> "Help me build an experiment that demonstrates nodeAffinity or pod anti-affinity, then we'll watch the scheduler's decision and verify it respected the rule. I want to type the commands myself — give them to me step by step and I'll verify each stage."

## Lab 5 — reconciliation

**To go deeper (the core of k8s):**
> "I deleted a pod manually and the ReplicaSet created a new one within seconds. Explain the full controller chain: Deployment controller → ReplicaSet controller → what exactly each one did here. What are owner references and how do they link them?"

**'edge vs level' challenge:**
> "Explain the difference between level-triggered and edge-triggered with a concrete scenario: what happens if the controller-manager crashes for 30 seconds while I delete a pod? Demonstrate it in practice if possible."

## Lab 6 — Build your own controller

**If you're stuck:**
> "I wrote tiny-controller.sh and it doesn't detect drift like I expected: [paste the script and the output]. Diagnose the logic."

**To go deeper — the leap to operators:**
> "My bash controller polls every 3 seconds. Explain how a real operator does this properly with informers and a work queue instead of polling, and why that matters for performance. Show me a minimal controller skeleton in Go with client-go, but explain every part — I want to understand, not just copy."

**CRD challenge:**
> "Help me define a simple CRD of my own (e.g. `Website`) and register it in the cluster, then we'll look in etcd at how it's stored exactly like a built-in resource. Walk me through it step by step."

---

# Part B — eBPF

> Reminder: this part requires a real Linux kernel. If you're on Mac/Windows, use Claude Code to set up a VM first (see below).

## Setting up the eBPF environment

**Rescuer prompt for setup (the single most useful use in the whole guide):**
> "I'm on [Mac/Windows/Ubuntu]. I need an environment where I can load eBPF programs — a real kernel with privileges. Check what I have and propose the simplest path (VM/Multipass/WSL2). Once we decide, install bpftrace, bpfcc-tools, and the matching headers, and verify everything works with `bpftrace --version` and a hello-world check. Explain each step."

**If bpftrace fails to load:**
> "bpftrace returns an error: [paste]. This is usually a kernel-headers or permissions issue. Diagnose and fix."

## Labs 7–8 — First bpftrace

**Important: type these one-liners yourself — they're short and instructive.** Use Claude Code only for understanding:

**To go deeper:**
> "I ran `bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[probe] = count(); }'`. Break this line down token by token: what a tracepoint is vs a kprobe, what `@` is, what `count()` is, and where exactly in the kernel this code runs. How is this different from running strace?"

**Build-a-one-liner challenge:**
> "I want a bpftrace one-liner that shows me the 10 processes doing the most disk reads. Don't give me the answer right away — ask me which tracepoint fits, and guide me to build it myself."

## Lab 9 — bpftrace script

**If you're stuck:**
> "I wrote netbytes.bt and it prints nothing / prints an error: [paste]. Diagnose — is this a tracepoint that doesn't exist on my kernel, or a syntax issue?"

**To go deeper:**
> "Explain what the verifier is and why my script passed/failed it. What are the constraints the verifier enforces — bounded loops, memory access, instruction count — and why each one exists."

## Lab 10 — BCC tools

**To go deeper (learning from existing code):**
> "Open the source of `execsnoop` from BCC. Walk me through both parts: the C program that runs in the kernel and the Python code that loads it and reads from the map. Show me exactly where the map is defined, where the kprobe is attached, and how the data crosses from the kernel to user space."

## Lab 11 — libbpf in C

**Here it's fine to let it write (tedious boilerplate):**
> "Help me build and run the XDP program that counts packets. Write the C file, compile it with clang, and load it with bpftool. But once it works — walk me through the code line by line and explain: what `SEC()` is, what CO-RE is, why `XDP_PASS`, and what would happen if I returned `XDP_DROP`."

**If the verifier rejects it:**
> "The verifier rejected my program with: [paste the verifier output]. Verifier output is notoriously scary — help me read it, understand which line the problem is on, and fix it. Explain what the verifier didn't like."

**Firewall challenge:**
> "Help me modify the XDP program so it drops (XDP_DROP) only ICMP (ping) packets. Guide me on how to parse the IP header inside eBPF, and how to verify it works with ping. I want to understand every line of the parsing."

---

# Part C — Cilium

## Lab 12 — Cilium + Hubble

**If you're stuck (Cilium install is complex):**
> "I installed Cilium but `cilium status` doesn't go green / there are pods in CrashLoopBackOff: [paste]. Diagnose — check the cilium agent logs and explain to me what went wrong."

**To go deeper:**
> "I now have a cluster with no kube-proxy — Cilium replaces it with eBPF. Explain exactly what happens when I create a Service: how the Cilium agent (which is a reconciliation controller) translates it into an eBPF map, and what the performance difference is versus classic kube-proxy iptables. Show me the map in practice with `cilium bpf lb list` if possible."

## Lab 13 — Network Policy

**To go deeper (the full connection):**
> "I applied a deny-all NetworkPolicy and saw drops in hubble. Explain the whole chain: from the NetworkPolicy (an object in the control plane, in etcd) → through the Cilium agent → to the eBPF program that drops the packet in the kernel. This is the intersection of the two topics I learned — connect them for me explicitly."

**L7 challenge:**
> "Help me write a CiliumNetworkPolicy at the L7 level that allows only `GET /public` and blocks `POST`. Explain how eBPF is able to enforce HTTP-level policy, and what it costs in performance versus a simple L3/L4 policy."

---

# Reusable prompt templates (copy-paste)

**When something breaks:**
> "The command `___` failed with the following output: [paste everything]. Don't guess — run diagnostic commands to find the root cause, explain to me what you found, and only then propose a fix. Confirm with me before running anything with sudo."

**For deep understanding after success:**
> "That worked. Now explain to me what happened under the hood line by line, and what would change if ___. Don't assume I understand — break it down to the concept level."

**To make sure you're learning and not just copying:**
> "Before you give me the solution — ask me 2–3 guiding questions that will help me reach it myself. I want to understand, not just make it work."

**For review and recap at the end of each part:**
> "I finished Part ___. Ask me 5 questions that test whether I really understood the key concepts, and correct me where I'm wrong."

---

## Summary recommendation

Building it with Claude Code is **very worthwhile** — as long as you stay the driver. It's strongest at environment setup (especially eBPF), at decoding scary verifier errors, and at being a 24/7 mentor for "why" questions. It's most dangerous to your learning when you let it run entire labs for you. Use the templates above, type the parts that teach yourself, and hand it the parts that are tedious — and you get a winning combination.
