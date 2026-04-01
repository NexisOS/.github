<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

# Overview

NexisOS is a free and open-source Linux distribution focused on **transparency, control, reproducibility, and high performance** through declarative configuration. Built around a custom package manager and a custom init system (`nexis-init`) using TOML files for system configuration, NexisOS takes a NixOS-inspired approach with fundamental design changes targeting faster rebuilds, native SELinux integration, pidfd-based process supervision, and compatibility with the standard Linux filesystem hierarchy.

Core components are written in **C** and built with **Meson + Ninja**.

---

## ⚠️ Project Status

NexisOS is in **early development and not yet in pre-alpha release**. Core architecture is still being designed, and no official binaries or installer ISOs are currently available. All features are experimental and intended for testing, evaluation, and development purposes only.

<div align="center">

| Build Status | Latest Stable Release |
|--------------|-----------------------|
| ![nexisos](https://img.shields.io/github/actions/workflow/status/NexisOS/nexisos/main.yml?label=nexisos) | ![GitHub release](https://img.shields.io/github/v/release/NexisOS/nexisos?label=latest%20stable) |

</div>

## 📥 Download

ISO builds will be available on SourceForge once the first release is ready:

[NexisOS on SourceForge](https://sourceforge.net/projects/nexisos/files/latest/download)

---

## 💡 Motivation: Why Not NixOS?

NixOS demonstrated that declarative, reproducible system management is viable at scale. NexisOS builds on that insight while rearchitecting the layers where NixOS introduces friction — particularly around rebuild performance, filesystem compatibility, and mandatory access control.

<details>
<summary>Specific NixOS pain points NexisOS addresses</summary>

| Pain Point | NixOS Behavior | NexisOS Approach |
| :--------- | :------------- | :--------------- |
| Rebuild cascades | Full derivation hashing couples store paths to the entire build environment, causing cascading rebuilds when any input changes | Two-level identity (BuildHash + InterfaceHash) — dependents only rebuild when the dependency's public ABI changes |
| SELinux incompatibility | Hashed store paths (`/nix/store/<hash>-...`) cannot be labeled with standard SELinux file-context rules | Projected FHS layout (`/usr/bin`, `/usr/lib`) receives standard SELinux labels at mount time |
| Evaluation speed | Nix language evaluation scales poorly on large module graphs | TOML configs are parsed directly — no DSL evaluation stage |
| FHS divergence | Non-standard paths break assumptions in third-party software, LD paths, and tooling | erofs generation images present a standard FHS; OverlayFS injection for `nexis shell` |
| Mutable state handling | Requires workarounds (e.g., `nix-ld`, `steam-run`) for software expecting writable or FHS-standard paths | Controlled mutability zones declared in TOML; FHS projection handles path expectations |
| Init / systemd compat | NixOS uses systemd, tightly coupling the init to systemd's assumptions. Non-systemd inits lose `sd_notify`, socket activation, and `systemctl` compatibility. | Custom `nexis-init` natively implements `sd_notify`, socket activation, and a `org.freedesktop.systemd1` D-Bus subset — applications and `systemctl` work without systemd as PID 1 |

</details>

---

## 🏗️ Key Design Principles

- **Declarative Configuration:** System and package management through TOML files with schema validation; no DSL required.
- **Graph-Aware Package Identity:** Two-level hashing (`BuildHash` for the package itself, `InterfaceHash` for dependent rebuild decisions) minimizes rebuild cascades to only what changed at the ABI boundary.
- **Atomic Rollbacks:** Generation symlink switching (O(1)) rather than full filesystem snapshots.
- **Immutable Store / erofs Projection:** Content-addressed store on ext4 with immutable objects; generations compiled into compressed erofs images mounted as the root filesystem (`/system`), carrying proper SELinux labels.
- **Native SELinux Integration:** SELinux is a first-class citizen; file contexts are baked into erofs images at generation build time.
- **pidfd-Based Init:** Custom PID 1 (`nexis-init`) written in C, built on a single-threaded `epoll` event loop with `pidfd` process supervision, `sd_notify` and socket activation protocol support, and a `org.freedesktop.systemd1` D-Bus compatibility layer via `sd-bus`.

---

## in-depth Performance Architecture

<details>
<summary>click to see</summary>

NexisOS targets measurably faster package operations than NixOS by reducing unnecessary work at each stage of the build and install pipeline.

<details>
<summary>View performance comparison table</summary>

| Operation | NixOS | NexisOS (Design Target) |
| :-------- | :---- | :---------------------- |
| **Content hashing** | SHA-256, single-threaded per file | BLAKE3, parallelized across cores via SIMD and multithreading |
| **Rebuild scope** | Full derivation hash — any environment change invalidates downstream | Two-level hashing — dependents only rebuild when InterfaceHash changes |
| **Store deduplication** | Post-hoc `nix-store --optimise` (global scan) | Insert-time deduplication — O(new files only) |
| **DAG traversal** | Evaluated in Nix language at rebuild time | Memory-mapped index over LMDB; zero-copy reads via mmap |
| **Build parallelism** | Jobserver-limited; sequential evaluation before build | Lock-free work-stealing queue; evaluation and build stages overlap where safe |
| **Filesystem activation** | Materializes symlink forests per generation | erofs image per generation — single `mount` activates the entire system |
| **Dev environment entry** | `nix-shell`: evaluates expression, builds derivation, creates gcroot (2–15s cached) | `nexis shell`: OverlayFS namespace injection over erofs base (<100ms cached) |
| **Boot: process supervision** | systemd: SIGCHLD + cgroup notification + PID tracking | `nexis-init`: pidfd via epoll — O(1) per exit event, zero signal handler overhead |
| **Boot: PID 1 footprint** | systemd: ~12 MB RSS, owns cgroups/journald/udev/logind | `nexis-init`: <2 MB RSS, single-threaded epoll loop, delegates to standalone tools |
| **Compression** | NAR format with xz/zstd | Hybrid chunking with Zstd (ratio) or LZ4 (speed) selected per content type |
| **Large store scaling** | Linear scans for GC and queries | LMDB mmap'd B+ tree for sub-linear lookups; optional bloom filters for content-hash existence checks |

> **Note:** These are architectural design targets. Benchmarks comparing equivalent operations against NixOS will be published as components reach testable maturity.

</details>

---

## 🔒 SELinux & Security

A core design goal is seamless SELinux support — something fundamentally difficult in NixOS due to hashed store paths that cannot match standard file-context patterns.

<details>
<summary>How NexisOS solves SELinux</summary>

1. **Projected FHS paths:** Runtime paths like `/usr/bin/ls` are presented via erofs generation images compiled from the CAS store. Standard SELinux `file_contexts` rules apply directly.
2. **Label-at-build:** SELinux labels are baked into the erofs image at generation build time via xattrs — no runtime labeling pass needed.
3. **Policy-as-config:** SELinux policy modules can be declared in TOML alongside package and service definitions, versioned and rolled back with the rest of the system.
4. **Immutable store isolation:** The `/store` CAS is not exposed to running processes directly. Only labeled, projected paths are visible in the process mount namespace.

```toml
# Example: declaring SELinux context for a service
[services.nginx]
exec = "/usr/sbin/nginx"
selinux.type = "httpd_t"
selinux.file_contexts = [
  { path = "/var/www(/.*)?", context = "httpd_sys_content_t" },
  { path = "/var/log/nginx(/.*)?", context = "httpd_log_t" },
]
```

</details>

<details>
<summary>Full security stack</summary>

| Layer | Implementation |
| :---- | :------------- |
| Firmware / Boot | Secure Boot, TPM 2.0, dm-verity |
| Kernel | SELinux (enforcing), seccomp, LKRG, IMA/EVM, lockdown mode |
| Runtime | `nexis-init` per-service sandboxing (namespaces, seccomp, SELinux transitions, capability drop), Tetragon (eBPF), ClamAV, LMD |
| Isolation | Mount namespaces, cgroups v2, per-service capability bounding and seccomp-BPF |
| Network | nftables, Suricata IDS/IPS |
| Filesystem | Immutable CAS on ext4; erofs generation images; declared mutability zones |

</details>

---

## ⚙️ Init System — `nexis-init`

NexisOS uses a custom PID 1 written in C, built on a single-threaded `epoll` event loop with `pidfd`-based process supervision. The design philosophy is: **implement the protocol interfaces that applications actually talk to, delegate everything else to proven standalone tools.**

<details>
<summary>Why not an existing init?</summary>

| Init | Limitation for NexisOS |
| :--- | :--------------------- |
| **systemd** | Monolithic; assumes ownership of cgroups, udev, journaling, networking, login sessions. Embedding NexisOS's declarative generation model inside systemd's assumptions would require fighting the tool. |
| **dinit** | Closest fit — dependency-based, lightweight. Lacks native pidfd support, no built-in cgroups v2 delegation, and no D-Bus interface for systemd-expecting applications. |
| **s6 / s6-rc** | Excellent supervision primitives, but the filesystem-as-database model conflicts with NexisOS's TOML-first configuration. No D-Bus systemd1 interface. |
| **OpenRC** | Shell-script-heavy, sequential by default. PID tracking via pidfiles is inherently racy — the exact problem pidfd solves. |
| **runit** | Minimal and robust for supervision, but no dependency ordering, no cgroups integration, no readiness protocol beyond file-descriptor passing. |

Every existing init would require a compatibility shim, a cgroups manager, and a D-Bus bridge bolted on top — at that point the shim layer is more complex than a purpose-built init.

</details>

<details>
<summary>Architecture diagram</summary>

```
┌─────────────────────────────────────────────────────────┐
│                nexis-init (PID 1)                       │
│                                                         │
│                epoll event loop                         │
│                                                         │
│    ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌───────────┐  │
│    │ pidfd   │ │ signalfd │ │ timerfd │ │ notify    │  │
│    │ (child  │ │ (SIGTERM │ │ (watch- │ │ socket    │  │
│    │  exits) │ │  SIGINT) │ │  dogs)  │ │ (sd_notify│  │
│    └────┬────┘ └────┬─────┘ └─────────┘ │  proto)   │  │
│         │           │                   └─────┬─────┘  │
│    ┌────┴───┐  ┌────┴──────┐            ┌─────┴─────┐  │
│    │ D-Bus  │  │ control   │            │ socket    │  │
│    │ socket │  │ socket    │            │ activation│  │
│    │(sd-bus)│  │(nexisctl) │            │ (LISTEN_  │  │
│    └────────┘  └───────────┘            │  FDS)     │  │
│                                         └───────────┘  │
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │  Service Supervisor  │  │  Dependency Graph Engine │ │
│  │  • clone3(CLONE_PIDFD│  │  • Topological sort      │ │
│  │  • cgroups v2 place  │  │  • Parallel startup      │ │
│  │  • namespace setup   │  │  • Before=/After=/       │ │
│  │  • SELinux transition│  │    Requires=/Wants=      │ │
│  │  • seccomp loading   │  │  • Target grouping       │ │
│  │  • capability drop   │  │  • Socket-activation     │ │
│  │  • restart policy    │  │    ordering              │ │
│  └──────────────────────┘  └──────────────────────────┘ │
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │  Unit File Parser    │  │  Compatibility Layer     │ │
│  │  • Reads systemd     │  │  • org.freedesktop.      │ │
│  │    [Unit], [Service], │  │    systemd1 D-Bus API   │ │
│  │    [Install], [Timer]│  │    (subset via sd-bus)   │ │
│  │  • Reads TOML service│  │  • sd_notify protocol    │ │
│  │    declarations      │  │  • LISTEN_FDS socket     │ │
│  │  • Both compile to   │  │    activation            │ │
│  │    same internal repr│  │  • /run/systemd/ state   │ │
│  └──────────────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

External standalone tools (not reimplemented):
  systemd-tmpfiles    → /tmp, /run directory setup
  systemd-sysusers    → system user/group creation
  systemd-sysctl      → kernel parameter application
  eudev / systemd-udevd → device management
  systemd-modules-load → kernel module loading
```

</details>

<details>
<summary>Core design: pidfd supervision</summary>

Traditional inits track child processes via `SIGCHLD` + `waitpid`, which is inherently racy — the PID can be reused between the signal delivery and the wait call. NexisOS uses `clone3()` with `CLONE_PIDFD` to get a stable file descriptor for each child process at spawn time.

```c
/* Simplified core loop structure */
int main(void) {
    int epfd = epoll_create1(EPOLL_CLOEXEC);

    /* Register all FD sources with epoll */
    int sig_fd  = signalfd(-1, &sigmask, SFD_CLOEXEC);
    int notify  = bind_notify_socket("/run/nexis/notify");
    int control = bind_control_socket("/run/nexis/control");
    int dbus_fd = sd_bus_get_fd(bus);

    epoll_ctl(epfd, EPOLL_CTL_ADD, sig_fd,  &(struct epoll_event){.events=EPOLLIN, .data.fd=sig_fd});
    epoll_ctl(epfd, EPOLL_CTL_ADD, notify,  &(struct epoll_event){.events=EPOLLIN, .data.fd=notify});
    epoll_ctl(epfd, EPOLL_CTL_ADD, control, &(struct epoll_event){.events=EPOLLIN, .data.fd=control});
    epoll_ctl(epfd, EPOLL_CTL_ADD, dbus_fd, &(struct epoll_event){.events=EPOLLIN, .data.fd=dbus_fd});

    struct epoll_event events[64];
    for (;;) {
        int n = epoll_wait(epfd, events, 64, -1);
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;
            if      (fd == sig_fd)  handle_signal(sig_fd);
            else if (fd == notify)  handle_sd_notify(notify, services);
            else if (fd == control) handle_control_command(control, services);
            else if (fd == dbus_fd) handle_dbus_message(bus, services);
            else                    handle_pidfd_exit(fd, services, epfd);
        }
    }
}
```

Each spawned service gets a pidfd registered with epoll:

```c
struct service_handle spawn_service(const struct service_config *svc, int epfd) {
    struct clone_args args = {
        .flags = CLONE_PIDFD,
        .pidfd = (uint64_t)&pidfd,
    };
    pid_t pid = syscall(SYS_clone3, &args, sizeof(args));

    if (pid == 0) {
        /* Child: set up sandbox before exec */
        enter_cgroup(svc->cgroup_path);
        apply_seccomp_filter(svc->seccomp_profile);
        set_selinux_context(svc->selinux_type);
        drop_capabilities(svc->ambient_caps);
        if (svc->namespaces)
            enter_namespaces(svc->namespaces);
        execve(svc->exec_path, svc->argv, svc->envp);
        _exit(127);
    }

    /* Register pidfd with epoll — notified exactly when this
       specific process exits, no PID-reuse races possible */
    epoll_ctl(epfd, EPOLL_CTL_ADD, pidfd,
              &(struct epoll_event){.events=EPOLLIN, .data.fd=pidfd});

    return (struct service_handle){ .pid=pid, .pidfd=pidfd, .config=svc };
}
```

When the pidfd becomes readable, the process has exited. No signal handler, no race condition, no scanning — just an epoll event on the exact file descriptor.

</details>

<details>
<summary>Compatibility strategy</summary>

**Standalone tools (work without systemd as PID 1):** NexisOS invokes these directly during boot rather than reimplementing them: `systemd-tmpfiles`, `systemd-sysusers`, `systemd-sysctl`, `systemd-modules-load`, `eudev`/`systemd-udevd`.

**Protocol interfaces implemented natively in the init:**

| Interface | What expects it | Implementation |
| :-------- | :-------------- | :------------- |
| **`sd_notify`** protocol | Services using `Type=notify` (PostgreSQL, nginx, etc.) | Unix datagram socket at `/run/nexis/notify`; parses `READY=1`, `STATUS=`, `MAINPID=`, `WATCHDOG=1`. ~200 lines. |
| **Socket activation** (`LISTEN_FDS`) | Services expecting pre-opened sockets | Init opens sockets, passes FDs to child via `LISTEN_FDS` + `LISTEN_PID` env vars. ~300 lines. |
| **`org.freedesktop.systemd1`** D-Bus API | `systemctl`, desktop environments, Cockpit, monitoring tools | Subset via `sd-bus`: `ListUnits`, `StartUnit`, `StopUnit`, `RestartUnit`, `GetUnit`, property queries. ~1000 lines for the 80% case. |
| **`/run/systemd/` state layout** | Tools that read PID files or check system state | Symlinked: `/run/systemd/notify` → `/run/nexis/notify`, state files written to expected paths. ~50 lines. |

</details>

<details>
<summary>Unit file support</summary>

`nexis-init` reads both TOML service declarations (native) and systemd unit files (compatibility). Both compile to the same internal representation:

```toml
# Native TOML declaration — /etc/nexis/services.toml
[services.nginx]
exec = "/usr/sbin/nginx"
type = "notify"
restart = "on-failure"
restart_sec = 5
requires = ["network-online.target"]
after = ["network-online.target"]
cgroup.memory_max = "512M"
cgroup.cpu_weight = 100
seccomp = "default"
selinux.type = "httpd_t"
namespaces = { net = false, ipc = true }
capabilities.ambient = ["NET_BIND_SERVICE"]
```

```ini
; systemd unit — also supported, parsed into same internal representation
[Unit]
Description=The NGINX HTTP Server
After=network-online.target
Requires=network-online.target

[Service]
Type=notify
ExecStart=/usr/sbin/nginx
Restart=on-failure
RestartSec=5
MemoryMax=512M
CPUWeight=100
```

Supported systemd unit directives cover the vast majority of real-world unit files (`[Unit]`, `[Service]`, `[Install]`, `[Timer]`, `[Socket]` sections). Unsupported directives are logged as warnings, not errors.

</details>

<details>
<summary>Boot sequence</summary>

```
kernel → nexis-init (PID 1)
  │
  ├── 1. Mount /proc, /sys, /dev, /run (tmpfs)
  ├── 2. Mount erofs generation image → /system
  ├── 3. Set hostname, clock
  ├── 4. systemd-modules-load          (standalone tool)
  ├── 5. systemd-sysctl                (standalone tool)
  ├── 6. systemd-sysusers              (standalone tool)
  ├── 7. systemd-tmpfiles --create     (standalone tool)
  ├── 8. eudev / systemd-udevd         (device manager)
  ├── 9. Mount remaining filesystems from /etc/fstab + TOML declarations
  ├── 10. Activate SELinux policy
  ├── 11. Start D-Bus system bus
  ├── 12. Resolve service dependency graph
  └── 13. Parallel service startup
          ├── network.target group
          ├── sockets.target group (pre-opened for socket activation)
          ├── basic.target group
          └── multi-user.target / graphical.target
```

</details>

<details>
<summary>C library dependencies</summary>

The init is deliberately conservative about dependencies to keep the binary small, auditable, and predictable:

| Library | Purpose | Why this one |
| :------ | :------ | :----------- |
| Linux `epoll` | Event loop | Direct syscall, no wrapper overhead, no runtime |
| `sd-bus` (from `systemd`) | D-Bus systemd1 API | Mature, well-tested C library; avoids pulling `libdbus` |
| Linux syscalls | `clone3`, `pidfd_open`, `signalfd`, `timerfd`, cgroups, namespaces, `mount` | Direct syscall wrappers via thin internal helpers |
| `toml-c` (or equivalent) | Parse TOML service declarations | Lightweight TOML parser for C |
| `syslog` | Logging to kernel ring buffer and syslog | Standard POSIX interface |

**Not used:** `libevent`/`libev` (unnecessary abstraction over epoll), `libcurl` (no HTTP needed in PID 1).

</details>

<details>
<summary><code>nexisctl</code> control interface</summary>

`nexisctl` is a separate binary that communicates with `nexis-init` over a Unix socket (`/run/nexis/control`). It also provides a `systemctl` compatibility shim:

```bash
nexisctl start nginx              # native control
nexisctl status nginx
nexisctl restart nginx
nexisctl list                     # list all services and states
nexisctl log nginx                # follow service output

# systemctl compatibility — symlink or wrapper
systemctl start nginx             # works identically (routes to nexisctl)
systemctl status nginx
systemctl enable nginx
```

</details>

<details>
<summary>Init security model</summary>

Every service spawned by `nexis-init` runs in a security sandbox by default:

| Layer | Mechanism | Default |
| :---- | :-------- | :------ |
| **Process isolation** | Linux namespaces (mount, PID, IPC, UTS, network) | mount + IPC namespaces enabled |
| **Capability restriction** | Ambient capabilities + bounding set | All capabilities dropped except those explicitly declared |
| **Syscall filtering** | seccomp-BPF profiles loaded before exec | Default restrictive profile |
| **MAC** | SELinux context transition at exec time | Enforced — service type declared in TOML or unit file |
| **Resource limits** | cgroups v2 (memory, CPU, IO, PIDs) | Default PID limit (4096), no memory/CPU limit unless declared |
| **Filesystem** | Read-only root (erofs), writable paths declared per-service | `/` is read-only; services get writable access only to declared paths |

This inverts the typical Linux init model where services run with full privileges unless explicitly sandboxed.

</details>

---

## 🛠️ Package Manager Architecture

<details>
<summary>View package installation flow</summary>

<div align="center" style="background-color: white; padding: 20px;">
  <img src="https://github.com/NexisOS/.github/blob/main/diagram1.svg" width="100%" alt="Package Install Flow">
</div>

```text
TOML Config (schema-validated)
        ↓
Dependency Graph Engine (incremental DAG)
        ↓
Package Builder (sandboxed, parallel)
        ↓
CAS Store (Merkle trees, insert-time dedup)
  /objects/<blake3>  +  /packages/<PackageID>/
        ↓
Metadata DB (LMDB — memory-mapped B+ tree, zero-copy reads,
             atomic transactions, no WAL tuning required)
        ↓
Generation Build (mkfs.erofs — compressed, immutable FHS image)
        ↓
Generation Activation (atomic symlink switch)
```

</details>

<details>
<summary>Graph-Aware Package Identity</summary>

Package identity uses a two-level hashing model to minimize unnecessary rebuilds:

```
BuildHash = blake3(source inputs + build script + dependency BuildHashes + build env metadata)

InterfaceHash = blake3(exported headers/symbols + ABI-relevant outputs + public API surface)
```

**BuildHash** determines whether a package itself needs rebuilding. **InterfaceHash** determines whether *dependents* need rebuilding. If a dependency's BuildHash changes but its InterfaceHash stays the same (e.g., an internal refactor that doesn't affect outputs), downstream packages are not rebuilt.

This is where NexisOS achieves its largest rebuild reduction compared to NixOS, where any change to a dependency's derivation hash forces all dependents to rebuild regardless of whether the output ABI changed.

</details>

<details>
<summary>Content-Addressed Store (CAS)</summary>

NexisOS uses a CAS for **storage and identity** but avoids using it as the runtime filesystem directly. The store is the source of truth; activation compiles packages into an immutable erofs image mounted as the root filesystem.

**Design:** Store in CAS → compile to erofs → mount.

```
/store/
    /objects/<blake3>            # immutable file objects (deduplicated)
    /packages/<PackageID>/       # logical package — FHS-structured tree
        manifest.toml
        files.list
        dependencies.list
```

Key properties: insert-time deduplication (no global optimization pass), LMDB metadata DB with zero-copy mmap reads, atomic copy-on-write transactions (crash-safe without WAL), and garbage collection via live reference set walk.

</details>

<details>
<summary>Generation Projection & Rollback</summary>

```
/generations/<id>/rootfs.erofs   # compressed, immutable FHS image
/system → /generations/<id>/     # atomic symlink to active generation
```

Each generation is a single **erofs image** compiled from the CAS via `mkfs.erofs`, containing the complete FHS tree with LZ4 compression and baked-in SELinux labels. Symlink switch enables **O(1) rollbacks**. Images are typically 500MB–2GB compressed and build in 1–5 seconds.

</details>

<details>
<summary>Filesystem Projection</summary>

NexisOS uses a **two-tier projection model**:

**Tier 1 — erofs (system activation):** At generation build time, all declared packages are composed into a single FHS tree, labeled with SELinux xattrs, then compiled into a compressed erofs image. One `mount` call activates the system — no layer limits, immutable by design, near-native read performance.

**Tier 2 — OverlayFS (ephemeral environments / `nexis shell`):** For sub-100ms activation, OverlayFS layers requested packages on top of the erofs base within a mount namespace.

| Use case | Mechanism | Latency | Layer limit concern |
|---|---|---|---|
| System generation | erofs image | 1–5s build, instant mount | None — single image |
| `nexis shell` | OverlayFS over erofs base | <100ms | No — base is 1 layer + few packages |
| Rollback | Remount different erofs image | <1ms | None |

</details>

<details>
<summary>Ephemeral Environments — <code>nexis shell</code></summary>

`nexis shell` replaces `nix-shell` with an ephemeral development environment built on **Linux mount namespaces + OverlayFS over the erofs system base**.

```bash
nexis shell python312 nodejs20 gcc
```

The tool resolves packages from CAS, creates a mount namespace via `unshare(CLONE_NEWNS)`, layers requested packages over the erofs base, and exports computed environment variables. **No files are copied, no symlink forests, no erofs rebuild.**

For reproducible project environments, `nexis-shell.toml` lockfiles work like `shell.nix`/`flake.nix` devShells:

```toml
# nexis-shell.toml — checked into version control
[shell]
packages = ["python@3.12.4", "nodejs@20.11", "postgresql@16"]
env.DATABASE_URL = "postgresql://localhost/devdb"

[shell.hooks]
enter = "echo 'Dev environment ready'"

[shell.services]
postgresql = { exec = "pg_ctl start -D $PGDATA", stop = "pg_ctl stop -D $PGDATA" }
```

`nexis shell --export-oci` produces an OCI container image from the shell environment.

</details>

---

## 🔒 Filesystem Access Policy

<details>
<summary>View directory mutability table</summary>

| Directory | Mutable | Notes |
| :-------- | :------ | :---- |
| `/store` | ❌ | Immutable content-addressed store |
| `/generations/<id>/rootfs.erofs` | ❌ | Compressed erofs image, SELinux labeled at build |
| `/system` | ❌ | Symlink to active generation |
| `/etc` | Partial | Three-layer merge: generation base, writable overlay for runtime state, `/etc/local/` for admin overrides |
| `/var/lib` | Partial | Application state; persistence declared per-service in TOML |
| `/var/log`, `/run`, `/tmp` | ✅ | Runtime and temporary data |
| `/home/<user>/.config`, `.local/share` | ❌ | Declared in TOML (user environment) |
| `/home/<user>/Documents`, `Downloads`, etc. | ✅ | User-controlled directories |

</details>

---

## 📁 Profiles & Configuration

NexisOS separates **configuration source** (what you write and version-control) from **runtime state** (what the tooling manages). Profiles are composable configuration layers that work identically on a single machine and across a fleet.

<details>
<summary>Configuration structure and profiles</summary>

### Configuration source — `/etc/nexis/`

```
/etc/nexis/                              # version-controlled
    system.toml                          # kernel, bootloader, locale, timezone, filesystem
    hardware.toml                        # auto-generated by installer, editable
    packages.toml                        # system-wide package declarations
    services.toml                        # service declarations and overrides
    users.toml                           # user accounts, groups, home environment

    profiles/                            # named, composable configuration layers
        workstation.toml
        server.toml
        gaming.toml
        development.toml

    modules/                             # reusable config fragments imported by profiles
        nginx.toml
        postgresql.toml
        monitoring.toml
```

Effective configuration is computed by merging base TOML files + active profiles + imported modules. Later layers override earlier ones with explicit errors for ambiguous merges.

### Activating profiles

```bash
nexis profile enable development gaming
nexis rebuild                              # builds new erofs generation with merged config
nexis profile list
nexis profile disable gaming
nexis profile diff +server -gaming         # preview changes without building
```

### Runtime state — `/nexis/`

```
/nexis/                                    # managed by tooling (not user-edited)
    db/
        metadata.lmdb                      # package metadata, content index
        generations.lmdb                   # generation history and profile snapshots
    generations/
        42/rootfs.erofs                    # compressed FHS image
        43/rootfs.erofs
    active -> 43                           # symlink to current generation
    cache/
        build/                             # intermediate build artifacts
        fetch/                             # downloaded sources, binary substitutes
```

</details>

---

## 🌐 Fleet Orchestration — `nexis fleet`

NexisOS includes built-in fleet management for deploying configurations across many machines from a single declaration, replacing tools like NixOps, colmena, and deploy-rs.

<details>
<summary>Fleet architecture, declaration, and commands</summary>

```
nexis-fleet.toml (fleet declaration)
        ↓
Fleet Resolver (computes per-host generation diffs)
        ↓
Distributed Build Mesh (builds across fleet spare capacity)
        ↓
Delta Sync (pushes only missing CAS objects per host)
        ↓
Atomic Activation (generation switch on each host)
```

Fleet **roles** resolve to **profiles** — the same abstraction used on single machines. The fleet resolver computes each host's effective config the same way, diffs against the running generation, and skips unchanged hosts entirely.

### Deployment strategies

| Strategy | Behavior |
|---|---|
| **rolling** | Activates hosts sequentially; stops on first failure; automatic rollback |
| **canary** | Deploys to a tagged subset first; waits for health checks; proceeds or aborts |
| **blue-green** | Builds full new generation across fleet; switches all hosts atomically |
| **parallel** | Deploys to all hosts simultaneously (for non-critical updates) |

### Fleet commands

```bash
nexis fleet plan                   # show what would change per host
nexis fleet deploy                 # execute with configured strategy
nexis fleet deploy --hosts web-01  # deploy to specific host(s)
nexis fleet rollback web-01        # rollback a specific host
nexis fleet status                 # generation, profiles, and health per host
nexis fleet diff web-01 web-02     # compare effective configs between hosts
```

</details>

</details>

---

## 🙏 Acknowledgments

<details>
<summary>Projects that inspired or support NexisOS</summary>

- **NixOS:** Declarative system configuration and reproducible builds
- **C ecosystem:** Language powering the package manager and init system
- **Meson + Ninja:** Build system providing fast, correct builds
- **sd-bus:** D-Bus library enabling the systemd1 compatibility layer
- **Security projects:** SELinux, ClamAV, R-FX Networks's LMD, Tetragon, Suricata, nftables
- The broader **Linux and open-source communities** for foundational tools and libraries

</details>

---

## 📢 Community & Support

As a personal project by a single maintainer, community channels are coming soon. For now, use [GitHub Issues](https://github.com/NexisOS/issues) for bugs, features, and ideas. GitHub Discussions and an official community forum/chat are planned.

---

## 📬 Contact

**Email**: [kyle.gortych.dev@gmail.com](mailto:kyle.gortych.dev@gmail.com) — Please use GitHub Issues for general inquiries. As the sole maintainer, response times may vary.

---

## ⚖️ Disclaimer

NexisOS is an independent, community-driven project currently maintained by an individual developer. It is not affiliated with the Linux Foundation, NixOS Foundation, or any other referenced projects. All trademarks belong to their respective owners and are used solely for identification.

The software is released under the GNU General Public License v3.0 and is provided **"as is," without warranty of any kind**, including any implied warranties of merchantability or fitness for a particular purpose, as described in the GPL license.

This project distributes **software code only**. The maintainer does not operate an online service, platform, or user account system and does not collect, process, or monitor end-user data through the distribution.

Certain jurisdictions may impose age-verification, identity, or monitoring requirements related to operating systems or online platforms. NexisOS does not currently implement such mechanisms, and no widely adopted cross-architecture standard exists for doing so across the diverse systems it supports.

Users, integrators, and redistributors are solely responsible for ensuring compliance with all applicable laws in their jurisdiction before installing, distributing, or deploying this software.

## Policy and Standards

NexisOS will be publishing an open letter requesting technical clarification and standards coordination regarding operating-system-level age assurance requirements affecting decentralized open-source systems to respective legislators, relevant regulatory and standards bodies.
