# Day 15 — Linux Internals (Container Fundamentals) Revision Notes

## 1. Why This Matters
- "A container is just a process" — containers aren't a special kernel feature, they're regular Linux processes given **isolation** (namespaces) and **resource limits** (cgroups) by the container runtime. Understanding this demystifies what Docker/containerd are actually doing under the hood.

## 2. Namespaces — What Actually Isolates a Container

Namespaces give a process its own **isolated view** of a global system resource — the process thinks it has the whole system to itself.

### PID Namespace
- Isolates process ID numbering. Inside a container, the main process sees itself as **PID 1** — even though on the host it has a completely different, real PID.
- **Why it matters**: PID 1 has special responsibilities in Linux (reaping zombie processes, handling signals) — this is why containers sometimes need an init process (like `tini`) if the main app doesn't handle signals properly as PID 1.

### Network Namespace
- Each container gets its own network stack: interfaces, routing table, IP addresses, iptables rules — completely separate from the host's and from other containers' network namespaces.
- This is the actual mechanism behind Docker's bridge networking (Day 2) — each container's `eth0` is a virtual interface pair (`veth`) with one end in the container's network namespace and the other attached to the host's bridge (`docker0`).

### Mount Namespace
- Isolates the filesystem mount points a process can see. This is what makes a container's filesystem look like a fresh root (`/`) even though it's really just a directory tree on the host (via overlay filesystems — see below).

### UTS Namespace
- Isolates hostname and domain name — this is why each container can have its own `hostname` (usually set to the container ID) independent of the host machine's actual hostname.

### Other Namespaces (know they exist)
- **IPC namespace**: Isolates System V IPC and POSIX message queues.
- **User namespace**: Maps container UID/GID to different (often unprivileged) UID/GID on the host — root inside a container can be mapped to a non-root user on the host, reducing the blast radius of a container escape.

### Interview Point
> "A container isn't a lightweight VM — it's a normal Linux process that the kernel gives an isolated view of PIDs, network, mounts, and hostname via namespaces. There's no hypervisor, no separate kernel — all containers on a host share the same kernel."

## 3. cgroups (Control Groups) — Resource Limiting

- While namespaces control **what a process can see**, cgroups control **how much of a resource a process (or group of processes) can use**.
- This is the actual mechanism behind Docker's `--memory`/`--cpus` flags and Kubernetes' `resources.limits` (Day 8).
- **cgroups v1 vs v2**: Modern kernels/container runtimes (containerd, recent Docker) default to **cgroups v2** — unified hierarchy, better resource accounting, improvements to how memory pressure is reported. Interview-relevant to know v2 is now the standard, v1 is legacy.
- **What it actually limits**: CPU shares/quota, memory (with OOM killing when exceeded — ties directly to Day 8's OOMKilled behavior), block I/O, network bandwidth (less commonly configured directly).
- **Interview point**: When a Pod hits its memory *limit*, the kernel's cgroup OOM killer terminates the process inside that cgroup — this is literally what "OOMKilled" in `kubectl describe pod` is reporting; it's a cgroups-level kernel event, not something Kubernetes itself decided arbitrarily.

## 4. Filesystem — OverlayFS
- Container images are built from **layers** (recall Day 2's Docker layer caching) — OverlayFS is the mechanism that stacks these read-only layers and adds a thin writable layer on top at runtime.
- **Copy-on-write**: When a running container modifies a file that exists in a lower read-only layer, OverlayFS copies it up into the writable layer first, then modifies the copy — the original image layers stay untouched (this is why the same image can run as multiple independent containers safely).

## 5. Container Runtime Landscape

### The Layers (top to bottom)
```
kubelet (Kubernetes node agent)
   ↓ talks via CRI (Container Runtime Interface)
containerd  OR  CRI-O
   ↓ talks via OCI runtime spec
runc (actually creates the namespaces/cgroups, starts the process)
```

### containerd
- A high-level container runtime, originally extracted from Docker, now a standalone CNCF project.
- Handles: image pulling/storage, container lifecycle, exposing the CRI so Kubernetes can talk to it directly.
- Most managed Kubernetes services (EKS, GKE) default to **containerd** as the node's container runtime today — Docker itself is no longer used as the direct Kubernetes runtime (Dockershim was removed in Kubernetes 1.24+).

### CRI-O
- Built specifically to be a minimal, Kubernetes-only CRI implementation (no extra Docker-compatible API surface) — used by Red Hat OpenShift by default.

### runc
- The low-level piece that actually does the OS-level work: creates namespaces, sets up cgroups, and executes the container process. Both containerd and CRI-O ultimately shell out to runc (or a runc-compatible OCI runtime) to do the real work.

### Why This Changed (interview-relevant history)
- Docker Engine itself was never designed to speak Kubernetes' CRI directly — Kubernetes used a shim (Dockershim) to translate. Once containerd matured and exposed CRI natively, Kubernetes dropped Dockershim (1.24+) — this is why "Docker support was removed from Kubernetes" caused confusion: **images built by Docker still work fine** (they're OCI-compliant), only the *daemon* Docker itself is no longer the node runtime.

---

## Quick Self-Test (do this without looking)
1. What's the actual difference between what a namespace does and what a cgroup does?
2. Why does the main process in a container see itself as PID 1, and what problem can that cause if the app doesn't handle it?
3. When Kubernetes reports "OOMKilled" for a Pod, what kernel-level mechanism is actually responsible for that?
4. Explain the chain from kubelet down to the actual container process being started — name each layer.
