# 🔱 Trishanku — Bare-Metal Linux Container Runtime & Observability Engine

[![C++20](https://img.shields.io/badge/C%2B%2B-20-00599C?logo=c%2B%2B&logoColor=white)](https://en.cppreference.com/w/cpp/20)
[![Linux Kernel](https://img.shields.io/badge/Linux_Kernel-Namespaces_%7C_Cgroups_v2-FCC624?logo=linux&logoColor=black)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
[![Bun](https://img.shields.io/badge/Bun-1.1+-000000?logo=bun&logoColor=white)](https://bun.sh/)
[![Next.js](https://img.shields.io/badge/Next.js-16_Turbopack-000000?logo=next.js&logoColor=white)](https://nextjs.org/)
[![Turborepo](https://img.shields.io/badge/Turborepo-Monorepo-EF4444?logo=turborepo&logoColor=white)](https://turbo.build/)
[![Alpine Linux](https://img.shields.io/badge/Alpine_Linux-mini--rootfs-0D597F?logo=alpinelinux&logoColor=white)](https://alpinelinux.org/)

**Trishanku** is a bare-metal Linux container runtime engineered from scratch in **C++20**, paired with a **Bun** orchestration daemon and a real-time **Next.js 16** telemetry dashboard within a polyglot Turborepo monorepo.

Standard container platforms (such as Docker, Podman, and containerd) treat the Linux kernel as a black box hidden behind layers of daemon abstraction. Trishanku pulls back the curtain, constructing isolated execution environments using raw Linux kernel primitives: `clone()` syscall flags, process namespaces, unified cgroups v2 hierarchies, and secure filesystem jailing with `pivot_root`.

---

## 📐 1. Architecture & System Flow

Trishanku spans three distinct architectural tiers: the low-level C++ runtime engine, the Bun asynchronous daemon, and the Next.js observability client.

```mermaid
flowchart TB
    subgraph Client ["Observability Layer (apps/web)"]
        UI["Next.js 16 App Router<br/>Terminal Console (xterm.js / stream)<br/>Live Metrics & Resource Gauges"]
    end

    subgraph Orchestrator ["Daemon Orchestrator (apps/daemon)"]
        Daemon["Bun Service (Port 8080)<br/>Bun.serve WebSocket Router<br/>OCI Execution Spec Builder<br/>Bun.spawn Process Supervisor"]
    end

    subgraph Runtime ["Low-Level Container Runtime (packages/runtime)"]
        Engine["Trishanku Binary (C++20)<br/>CLI Arguments & OCI Spec Ingestion<br/>Stack Allocation (1MB)"]
        
        subgraph Kernel ["Linux Kernel Primitives"]
            NS["Namespaces<br/>CLONE_NEWPID (PID 1)<br/>CLONE_NEWUTS (hostname)<br/>CLONE_NEWNS (mount private)"]
            CG["Cgroups v2 (/sys/fs/cgroup/trishanku/)<br/>memory.max | cpu.max | cgroup.procs"]
            FS["RootFS Jailing<br/>pivot_root() | MS_BIND<br/>Fresh /proc Mount | Unlink .old_root"]
            Payload["Isolated Container Payload<br/>(/bin/sh inside Alpine Linux rootfs)"]
        end
    end

    UI <-->|"WebSocket Telemetry (Bidirectional Streams)"| Daemon
    Daemon -->|"Bun.spawn(['sudo', './bin/trishanku', ...])"| Engine
    Engine -->|"1. clone(child_fn, stack, flags, ...)"| NS
    Engine -->|"2. Write PID to cgroup.procs & apply limits"| CG
    Engine -->|"3. pivot_root(rootfs, old_root)"| FS
    FS --> Payload
    Payload -->|"Raw stdout / stderr"| Engine
    Engine -->|"Stream stdout / stderr"| Daemon
    Daemon -->|"Broadcast Log Chunks & Cgroup Metrics"| UI
```

---

## 🧱 2. Core Technical Pillars & Kernel Primitives

### A. Linux Namespaces (Process Isolation)
The Linux kernel uses namespaces to partition global system resources so that processes see only their isolated slice of the system:
* **`CLONE_NEWPID`**: Isolates process IDs. The container's root entry process becomes `PID 1` inside its namespace, taking responsibility for child process harvesting, while carrying an unprivileged host PID.
* **`CLONE_NEWUTS`**: Isolates hostname and NIS domain identities via `sethostname("trishanku-sandbox", ...)`.
* **`CLONE_NEWNS`**: Clones the filesystem mount tree into a private namespace. Mounts are explicitly marked private (`mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL)`) to prevent mount changes from propagating to host namespaces.

### B. Control Groups v2 (Resource Enforcement)
Trishanku enforces strict resource ceilings through the unified cgroups v2 tree (`/sys/fs/cgroup`):
* Dynamically provisions a container leaf cgroup: `/sys/fs/cgroup/trishanku/<container_id>/`.
* Migrates the target PID by writing to `/sys/fs/cgroup/trishanku/<container_id>/cgroup.procs`.
* Enforces hard memory boundaries via `memory.max` (e.g., `268435456` for 256MB) and throttles CPU cycles via `cpu.max` quotas.

### C. Root Filesystem Jailing (`pivot_root`)
Unlike `chroot`—which does not change the root mount and is susceptible to escape via file descriptor hopping or `fchdir()`—Trishanku implements the atomic `pivot_root` sequence:
1. Bind-mounts the extracted Alpine Linux mini-rootfs onto itself (`MS_BIND | MS_REC`).
2. Creates a temporary detachment target directory: `<new_root>/.old_root`.
3. Issues `pivot_root(new_root, old_root)`.
4. Mounts a isolated pseudo-filesystem at `/proc` to guarantee process introspection commands (`ps`, `top`) only reflect the container namespace.
5. Detaches and unmounts the host root via `umount2("/.old_root", MNT_DETACH)` and removes the directory placeholder.

### D. Daemon Orchestrator & Live Telemetry (Bun)
* Serves HTTP and WebSocket endpoints using Bun's native C-level networking.
* Translates API container configurations into execution bundles.
* Invokes the compiled C++ binary using `Bun.spawn()` and captures standard input, output, and error streams as standard `ReadableStream` instances.
* Polls `/sys/fs/cgroup/trishanku/<id>/memory.current` and `cpu.stat` to stream telemetry frames directly to the browser.

---

## 📁 3. Monorepo Organization

```text
trishanku/
├── apps/
│   ├── daemon/                    # Bun + TypeScript process orchestrator & telemetry server
│   │   ├── index.ts               # Bun.serve HTTP/WebSocket router & spawn manager
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/                       # Next.js 16 + Tailwind CSS observability dashboard
│       ├── src/
│       │   └── app/               # App Router pages (Terminal, Container Cards, Metrics)
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── runtime/                   # Low-level C++20 container runtime engine
│   │   ├── src/
│   │   │   └── main.cpp           # Kernel clone(), namespaces, cgroups v2, pivot_root
│   │   ├── rootfs/                # Target directory for unpacked Alpine mini-rootfs
│   │   ├── Makefile               # Native g++ build targets (build, run, clean)
│   │   └── package.json           # Turborepo npm bridge ("build": "make build")
│   ├── eslint-config/             # Shared ESLint configuration
│   └── typescript-config/         # Shared TypeScript configuration
├── package.json                   # Root workspace manifest
├── pnpm-workspace.yaml            # Monorepo workspace boundaries
└── turbo.json                     # Turborepo task pipeline orchestration
```

---

## 🗺️ 4. Implementation Roadmap

| Phase | Milestone | Core Components |
|---|---|---|
| **Phase 1** | **Process Namespace Isolation** | `clone()` with `CLONE_NEWPID \| CLONE_NEWUTS \| CLONE_NEWNS`, child stack setup, PID 1 shell verification |
| **Phase 2** | **Filesystem Jailing & RootFS** | Alpine Linux mini-rootfs extraction, `pivot_root` sequence, fresh `/proc` mount, detachment |
| **Phase 3** | **Cgroups v2 Resource Throttling** | Dynamic `/sys/fs/cgroup/trishanku/<id>` leaf nodes, `memory.max`, `cpu.max`, synthetic OOM tests |
| **Phase 4** | **Bun Daemon Orchestration** | `Bun.serve` WebSocket hub, `Bun.spawn()` process manager, stream multiplexing |
| **Phase 5** | **Next.js Observability UI** | Real-time terminal streaming, active container status, live memory & CPU graph cards |

---

## ⚡ 5. Prerequisites & Quickstart

### Prerequisites

* **Linux OS** with Linux Kernel 5.8+ (Cgroups v2 enabled)
* **GCC / G++ 12+** (supporting `-std=c++20`)
* **Make**
* **Bun 1.1+**
* **pnpm 9+** / **Node.js 20+**
* Root privileges (`sudo`) for Linux namespace and cgroup manipulation

### Building the Runtime

```bash
# Clone the repository
git clone https://github.com/avinashk47/trishanku.git
cd trishanku

# Install monorepo dependencies
pnpm install

# Compile the C++20 container runtime via Turborepo
pnpm run build

# Or compile directly in packages/runtime
cd packages/runtime && make build
```

> [!IMPORTANT]
> Linux container primitives (`clone()` with new namespaces, mounting filesystems, and writing to `/sys/fs/cgroup/`) require `root` capabilities or `sudo`.

---

## 🔬 6. Verification & Testing

* **PID Isolation Check**: Running `echo $$` inside the spawned container shell yields `PID 1`.
* **Hostname Isolation Check**: Changing hostname via `hostname trishanku-test` does not alter the host system hostname.
* **Mount Tree Isolation Check**: Host mount table remains untouched when mounts/unmounts occur inside the container.
* **Cgroup Memory Cap Verification**: Stress tests allocating beyond `memory.max` trigger clean kernel OOM termination without impacting host memory stability.
