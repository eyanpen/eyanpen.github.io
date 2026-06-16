# Runtime Backends: A Deep Dive into qwrap vs Container Isolation Modes

> In sandbox runtimes, "isolation" is the core requirement. qwrap (based on bwrap user namespace) and Container (podman/docker) are two mainstream backends. They solve the same problem — running code in a restricted environment — but take completely different paths. This article uses extensive analogies to help you understand the similarities and differences.

## Building Intuition: Two Ways to "Lock the Door"

Imagine you need to confine someone you don't fully trust in a room to do work:

- **qwrap approach**: In your existing house, you put up a partition to wall off a corner, leaving only a small window to pass materials through. The walls are still the original walls, the floor is still the original floor, but the person can only see what's inside the partition.

- **Container approach**: You build a shipping container with its own independent power, water, and ventilation systems. Put the person inside, close the door. They feel like they're in a complete little house, completely unaware of what's outside.

This is the most fundamental difference: **qwrap is lightweight view isolation, Container is complete environment encapsulation**.

## What is qwrap (bwrap user namespace)

qwrap uses `bubblewrap` (bwrap) under the hood, a sandboxing tool that leverages Linux user namespaces.

### How it Works

```
Host filesystem
├── /usr/bin/python3          ← Host's Python
├── /home/user/project/       ← User project
└── /tmp/secrets/             ← Sensitive files

qwrap sandbox view (what the process sees)
├── /usr/bin/python3          ← bind-mounted in, read-only
├── /workspace/               ← Only the project directory is exposed
└── (/tmp/secrets/ doesn't exist) ← Completely invisible
```

Key mechanisms:
- **User Namespace**: The process thinks it's root, but actually maps to an unprivileged user on the host
- **Mount Namespace**: Only bind-mounts necessary directories in, everything else is invisible
- **No images, no layers, no network namespace** (unless explicitly configured)

### Analogy: VPN Split Tunneling

qwrap is like split-tunneling rules on your phone — you're not wrapping the entire phone in a VPN, just routing specific apps through the proxy. The system is still the same system, just with a restricted "field of view."

## What is Container (podman/docker)

A Container is a complete isolated runtime environment, using multiple Linux namespaces (pid, net, mount, uts, ipc) plus cgroups for resource limits.

### How it Works

```
Host
└── Running podman/docker daemon (or rootless direct fork)

Container interior
├── /usr/bin/python3          ← Shipped with the image, may differ from host version
├── /workspace/               ← Volume mounted in
├── Independent PID 1         ← Process tree starts from 1
├── Independent network stack ← Has its own eth0, IP address
└── Independent hostname      ← Not the host's name
```

Key mechanisms:
- **OCI Image**: Environment fully packaged, including OS base layer, dependency libraries, toolchain
- **Multi-dimensional Namespaces**: PID, network, mount, hostname all isolated
- **Cgroups**: CPU, memory, IO can be capped
- **Layered filesystem**: OverlayFS, writes don't affect the base image

### Analogy: A "Poor Man's VM"

A Container is like a "lightweight virtual machine" — without the overhead of hardware virtualization, but giving the process an experience nearly equivalent to owning a dedicated machine.

## Core Differences

### Startup Speed

- **qwrap**: Millisecond-level. Essentially just `clone()` + set up a few namespaces + exec, similar to starting a regular process.
- **Container**: Hundreds of milliseconds to seconds. Needs to prepare rootfs (extract layers/mount overlay), configure networking, start init process.

Example: You have an AI Agent that repeatedly executes user-submitted Python snippets, each needing isolation. With Container, doing `docker run` then `docker rm` each time becomes unsustainable at one call per second. qwrap can launch dozens of sandbox instances per second.

### Isolation Strength

- **qwrap**: Medium. The process still shares the host kernel, network is not isolated by default (can access the internet), only filesystem view trimming and privilege reduction.
- **Container**: Strong. Network, PID, and filesystem are comprehensively isolated. Combined with a seccomp profile, even syscalls can be restricted.

Example: If sandboxed code attempts `kill -9 1` (kill the init process):
- qwrap: Since it lacks CAP_KILL privileges over host processes in the user namespace, the kernel rejects the operation, but the process can "see" host PIDs (unless PID namespace is added).
- Container: PID 1 as seen by the process is the container's own init — killing it only crashes the container itself, the host is unharmed.

### Environment Consistency

- **qwrap**: Depends on the host environment. If the host doesn't have `numpy` installed, the sandbox doesn't either (unless you mount a virtualenv directory in).
- **Container**: Self-contained environment. Whatever is installed in the image is available, regardless of what's on the host.

Example: Your CI runs on an Ubuntu 22.04 machine, but the project needs Python 3.12 + CUDA 12.
- qwrap approach: You must install Python 3.12 and CUDA on the host first, then qwrap just restricts visible scope.
- Container approach: Simply `FROM nvidia/cuda:12.0-python3.12`, everything is in the image, even if the host is CentOS 7.

### Resource Overhead

- **qwrap**: Near-zero overhead. No extra processes, no overlay filesystem, no virtual bridge. The sandbox is just a "process with a restricted view."
- **Container**: Lightweight but perceptible. Each container has its own mount stack, possibly a veth pair, and a cgroup controller tracking it. Running a few is fine, but running hundreds starts accumulating network and storage overhead.

### Portability

- **qwrap**: Linux-only (depends on user namespace), requiring kernel version ≥ 3.8. Different distributions have different user namespace policies (Ubuntu enables by default, Debian/RHEL may need sysctl adjustments).
- **Container**: Cross-platform. macOS/Windows can run them through VM layers (Docker Desktop, Podman Machine). Images are standard OCI format, deployable anywhere.

## When to Choose qwrap

- Need extremely fast startup/teardown cycles (Agent spawns a sandbox for every tool call)
- Host environment is already prepared, just need to "restrict visibility"
- Don't need network isolation (or willing to manage with iptables manually)
- Resource-sensitive, don't want extra memory/storage overhead for isolation
- Runtime environment is definitely Linux with user namespace support

Typical scenario: **Code execution sandbox**. An AI coding assistant runs LLM-generated code in qwrap, discards it when done. May run dozens of times per second, each needing only a Python interpreter + limited file access.

## When to Choose Container

- Need a complete, reproducible runtime environment (the "works on my machine" problem disappears)
- Need strong isolation (untrusted code, multi-tenant scenarios)
- Need network isolation (each task gets an independent network stack)
- Need cross-platform deployment
- Longer lifecycle (service processes, long-running tasks)

Typical scenario: **CI/CD Pipeline**. Each build runs in a clean container ensuring environment consistency. Or **multi-tenant SaaS**, where each tenant's custom logic runs in an isolated container with full resource and network separation.

## Can You Combine Them?

Yes, and this is a very common pattern:

- **Outer Container + Inner qwrap**: Container provides environment consistency and coarse-grained isolation, qwrap provides fine-grained per-process sandboxing inside the container. For example, an Agent service runs inside a container, and each tool invocation spawns a qwrap sandbox.

- **qwrap as a "lightweight container" substitute**: In development environments where you don't want to install Docker but need some isolation, qwrap can serve as a minimal alternative.

## Summary at a Glance

- Startup latency: qwrap milliseconds / Container hundreds of ms to seconds
- Isolation dimensions: qwrap filesystem + user privileges / Container filesystem + network + PID + resources
- Environment dependency: qwrap depends on host / Container self-contained image
- Resource overhead: qwrap near-zero / Container lightweight but perceptible
- Portability: qwrap Linux-only / Container cross-platform
- Best for: qwrap high-frequency short-lived / Container long-lived + strong isolation

## Conclusion

Choosing between qwrap and Container is fundamentally a tradeoff between "light" and "complete":

- If you want to "quickly blindfold a process" — choose qwrap
- If you want to "lock a process in an independent shipping container" — choose Container

Understanding this distinction, you can make sound layered decisions when designing sandbox systems: use Containers to solve environment consistency, use qwrap to solve high-frequency isolated execution, and combine both to cover everything from CI to Agent Runtime.
