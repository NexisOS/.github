<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

# 🐧 Overview

NexisOS is a free and open-source Linux distribution focused on **transparency, control, reproducibility, and high performance** through declarative configuration. Built around a custom Rust-based package manager and a custom Rust-based init system (`nexis-init`) using TOML configuration files, NexisOS takes a NixOS-inspired approach with fundamental design changes targeting faster rebuilds, native SELinux integration, pidfd-based process supervision, and compatibility with the standard Linux filesystem hierarchy.

---

## ⚠️ Project Status

NexisOS is in **early development and not yet in pre-alpha release**. Core architecture is still being designed, and no official binaries or installer ISOs are currently available. All features are experimental and intended for testing, evaluation, and development purposes only.

---

## 💡 Motivation: Why Not NixOS?

NixOS demonstrated that declarative, reproducible system management is viable at scale. NexisOS builds on that insight while rearchitecting the layers where NixOS introduces friction particularly around rebuild performance, filesystem compatibility, and mandatory access control.

<details>
<summary>Specific NixOS pain points NexisOS addresses</summary>

| Pain Point | NixOS Behavior | NexisOS Approach |
| :--------- | :------------- | :--------------- |
| Rebuild cascades | Full derivation hashing couples store paths to the entire build environment, causing cascading rebuilds when any input changes | Two-level identity (BuildHash + InterfaceHash) — dependents only rebuild when the dependency's public ABI changes, not on every internal modification |
| SELinux incompatibility | Hashed store paths (`/nix/store/<hash>-...`) cannot be labeled with standard SELinux file-context rules | Projected FHS layout (`/usr/bin`, `/usr/lib`) receives standard SELinux labels at mount time |
| Evaluation speed | Nix language evaluation scales poorly on large module graphs | TOML configs are parsed directly — no DSL evaluation stage |
| FHS divergence | Non-standard paths break assumptions in third-party software, LD paths, and tooling | erofs generation images present a standard FHS; OverlayFS injection for `nexis shell` |
| Mutable state handling | Requires workarounds (e.g., `nix-ld`, `steam-run`) for software expecting writable or FHS-standard paths | Controlled mutability zones declared in TOML; FHS projection handles path expectations |
| Rebuild diagnostics | Identifying the root cause of a rebuild failure often requires manual graph inspection | Incremental DAG tracks per-node status; failures report the originating dependency directly |
| Init / systemd compat | NixOS uses systemd, tightly coupling the init to systemd's assumptions. Non-systemd inits (s6, dinit) lose `sd_notify`, socket activation, and `systemctl` compatibility. | Custom `nexis-init` natively implements `sd_notify`, socket activation, and a `org.freedesktop.systemd1` D-Bus subset — applications and `systemctl` work without systemd as PID 1 |

</details>

---

## 🏗️ Key Design Principles

- **Declarative Configuration:** System and package management through TOML files with schema validation; no DSL required.
- **Graph-Aware Package Identity:** Two-level hashing (`BuildHash` for the package itself, `InterfaceHash` for dependent rebuild decisions) enables binary reuse and minimizes rebuild cascades to only what changed at the ABI boundary.
- **Atomic Rollbacks:** Generation symlink switching (O(1)) rather than full filesystem snapshots.
- **Immutable Store / erofs Projection:** Content-addressed store on ext4 with immutable objects; generations compiled into compressed erofs images mounted as the root filesystem (`/system`), carrying proper SELinux labels.
- **Native SELinux Integration:** SELinux is a first-class citizen, not a bolt-on; file contexts are baked into erofs images at generation build time.
- **Security-First:** Pre-configured security stack from firmware through network layer.
- **pidfd-Based Init:** Custom PID 1 (`nexis-init`) built on a single-threaded `mio`/`epoll` event loop with `pidfd` process supervision, `sd_notify` and socket activation protocol support, and a `org.freedesktop.systemd1` D-Bus compatibility layer via `zbus` — maximizing application compatibility with minimal code.

---

## ⚡ Performance Architecture

NexisOS targets measurably faster package operations than NixOS by reducing unnecessary work at each stage of the build and install pipeline.

<details>
<summary>View performance comparison table</summary>

| Operation | NixOS | NexisOS (Design Target) |
| :-------- | :---- | :---------------------- |
| **Content hashing** | SHA-256, single-threaded per file | BLAKE3, parallelized across cores via SIMD and multithreading |
| **Rebuild scope** | Full derivation hash — any environment change invalidates downstream | Two-level hashing — dependents only rebuild when InterfaceHash changes, not on every BuildHash change |
| **Store deduplication** | Post-hoc `nix-store --optimise` (global scan) | Insert-time deduplication — O(new files only), no store-wide scans |
| **DAG traversal** | Evaluated in Nix language at rebuild time | Memory-mapped index over LMDB; zero-copy reads via mmap |
| **Build parallelism** | Jobserver-limited; sequential evaluation before build | Lock-free work-stealing queue; evaluation and build stages overlap where safe |
| **Filesystem activation** | Materializes symlink forests per generation | erofs image per generation — single `mount` activates the entire system |
| **Generation image build** | N/A (symlink forest creation) | `mkfs.erofs` composes FHS tree with LZ4 compression (1–5s for full system) |
| **Dev environment entry** | `nix-shell`: evaluates expression, builds derivation, creates gcroot (2–15s cached) | `nexis shell`: OverlayFS namespace injection over erofs base (<100ms cached) |
| **Boot: process supervision** | systemd: SIGCHLD + cgroup notification + PID tracking | `nexis-init`: pidfd via epoll — O(1) per exit event, zero signal handler overhead |
| **Boot: service startup** | systemd: generator pass + unit graph evaluation at boot | `nexis-init`: in-memory DAG with topological sort at load, parallel startup with socket activation |
| **Boot: PID 1 footprint** | systemd: ~12 MB RSS, owns cgroups/journald/udev/logind | `nexis-init`: <2 MB RSS, single-threaded mio event loop, delegates to standalone tools |
| **Fleet rebuilds** | NixOps/colmena: sequential host evaluation, per-host builds | Distributed build mesh with work-stealing across fleet; shared CAS deduplicates cross-host |
| **Compression** | NAR format with xz/zstd | Hybrid chunking with Zstd (ratio) or LZ4 (speed) selected per content type |
| **Large store scaling** | Linear scans for GC and queries | LMDB mmap'd B+ tree for sub-linear lookups; optional bloom filters for content-hash existence checks |

> **Note:** These are architectural design targets. Actual performance will depend on implementation quality and workload characteristics. Benchmarks comparing equivalent operations against NixOS will be published as components reach testable maturity.

</details>

---

## 🔒 SELinux & Mandatory Access Control

A core design goal of NexisOS is seamless SELinux support — something fundamentally difficult in NixOS due to hashed store paths that cannot match standard file-context patterns.

<details>
<summary>How NexisOS solves this</summary>

1. **Projected FHS paths:** Runtime paths like `/usr/bin/ls` and `/usr/lib/libc.so` are presented via erofs generation images compiled from the CAS store. These paths are stable and predictable, so standard SELinux `file_contexts` rules apply directly.

2. **Label-at-build:** SELinux labels are baked into the erofs image at generation build time. The `mkfs.erofs` step reads xattrs from the assembled FHS tree, so labels are present from the moment the image is mounted — no runtime labeling pass needed.

3. **Policy-as-config:** SELinux policy modules can be declared in TOML alongside package and service definitions, versioned and rolled back with the rest of the system.

4. **Immutable store isolation:** The `/store` CAS is not exposed to running processes directly. Only the labeled, projected paths are visible in the process mount namespace, reducing the attack surface.

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

---

## 🔒 Security Stack

<details>
<summary>View full security stack</summary>

| Layer | Implementation |
| :---- | :------------- |
| Firmware / Boot | Secure Boot, TPM 2.0, dm-verity |
| Kernel | SELinux (enforcing), seccomp, LKRG, IMA/EVM, lockdown mode |
| Runtime | `nexis-init` per-service sandboxing (namespaces, seccomp, SELinux transitions, capability drop), Tetragon (eBPF runtime security), ClamAV, LMD (R-FX Networks) |
| Isolation | `nexis-init` mount namespaces, cgroups v2, per-service capability bounding and seccomp-BPF |
| Network | nftables, Suricata IDS/IPS |
| Filesystem | Immutable CAS on ext4; erofs generation images; declared mutability zones |

</details>

---

## ⚙️ Init System `nexis-init`

NexisOS uses a custom PID 1 written in Rust, built on a single-threaded `mio`/`epoll` event loop with `pidfd`-based process supervision. The design philosophy is: **implement the protocol interfaces that applications actually talk to, delegate everything else to proven standalone tools.**

<details>
<summary>Why not an existing init?</summary>

| Init | Limitation for NexisOS |
| :--- | :--------------------- |
| **systemd** | Monolithic; assumes ownership of cgroups, udev, journaling, networking, login sessions, and more. Embedding NexisOS's declarative generation model inside systemd's assumptions would require fighting the tool rather than using it. |
| **dinit** | Closest fit — dependency-based, lightweight, well-written C++. Lacks native pidfd support, no built-in cgroups v2 delegation, and no D-Bus interface for systemd-expecting applications. Would still need a compatibility shim layer on top. |
| **s6 / s6-rc** | Excellent supervision primitives, but the filesystem-as-database model conflicts with NexisOS's TOML-first configuration. No D-Bus systemd1 interface. Adding compatibility on top would exceed the complexity of a purpose-built init. |
| **OpenRC** | Shell-script-heavy, sequential by default. PID tracking via pidfiles is inherently racy — the exact problem pidfd solves. |
| **runit** | Minimal and robust for supervision, but no dependency ordering, no cgroups integration, no readiness protocol beyond file-descriptor passing. |

The common thread: every existing init would require a compatibility shim, a cgroups manager, and a D-Bus bridge bolted on top. At that point the shim layer is more complex than a purpose-built init that natively speaks the right protocols.

</details>

<details>
<summary>Why mio, not tokio</summary>

PID 1 is the one process that must never crash and must never allocate unexpectedly. tokio brings a multi-threaded work-stealing runtime, a global task scheduler, and allocator pressure from task spawning — all liabilities for the process that every other process depends on.

`mio` is a thin wrapper over `epoll` (or `io_uring` on supported kernels). The init's event loop is a single `epoll_wait` call multiplexing all file descriptors: pidfds, signalfd, timerfd, notify sockets, D-Bus connections, and the control socket. No thread pool, no executor, no implicit allocations. The init's memory footprint is bounded and predictable.

</details>

<details>
<summary>Architecture diagram</summary>

```
┌─────────────────────────────────────────────────────────┐
│                nexis-init (PID 1)                       │
│                                                         │
│                                                         │
│                mio epoll event loop                     │
│                                                         │
│    ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌───────────┐   │
│    │ pidfd   │ │ signalfd │ │ timerfd │ │ notify    │   │
│    │ (child  │ │ (SIGTERM │ │ (watch- │ │ socket    │   │
│    │  exits) │ │  SIGINT) │ │  dogs)  │ │ (sd_notify│   │
│    └────┬────┘ └────┬─────┘ └─────────┘ │  proto)   │   │
│         │           │                   └─────┬─────┘   │
│    ┌────┴───┐  ┌────┴──────┐            ┌─────┴─────┐   │
│    │ D-Bus  │  │ control   │            │ socket    │   │
│    │ socket │  │ socket    │            │ activation│   │
│    │(zbus)  │  │(nexisctl) │            │ (LISTEN_  │   │
│    └────────┘  └───────────┘            │  FDS)     │   │
│                                         └───────────┘   │
│                                                         │
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │  Service Supervisor  │  │  Dependency Graph Engine │ │
│  │                      │  │                          │ │
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
│  │                      │  │                          │ │
│  │  • Reads systemd     │  │  • org.freedesktop.      │ │
│  │    [Unit], [Service],│  │    systemd1 D-Bus API    │ │
│  │    [Install], [Timer]│  │    (subset via zbus)     │ │
│  │  • Reads TOML service│  │  • sd_notify protocol    │ │
│  │    declarations      │  │  • LISTEN_FDS socket     │ │
│  │  • Both compile to   │  │    activation            │ │
│  │    same internal repr│  │  • /run/systemd/ state   │ │
│  │                      │  │    directory layout      │ │
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

```rust
// Simplified core loop structure
fn main() -> Result<()> {
    let mut poll = mio::Poll::new()?;
    let mut events = mio::Events::with_capacity(256);

    // Register all FD sources with epoll via mio
    let signal_fd = SignalFd::new(&[SIGTERM, SIGINT, SIGCHLD])?;
    let notify_sock = NotifySocket::bind("/run/nexis/notify")?;
    let control_sock = ControlSocket::bind("/run/nexis/control")?;
    let dbus_conn = DBusConnection::system()?;

    poll.registry().register(&signal_fd, SIGNAL, Interest::READABLE)?;
    poll.registry().register(&notify_sock, NOTIFY, Interest::READABLE)?;
    poll.registry().register(&control_sock, CONTROL, Interest::READABLE)?;
    poll.registry().register(&dbus_conn, DBUS, Interest::READABLE)?;

    loop {
        poll.poll(&mut events, None)?;
        for event in events.iter() {
            match event.token() {
                SIGNAL   => handle_signal(&signal_fd)?,
                NOTIFY   => handle_sd_notify(&notify_sock, &mut services)?,
                CONTROL  => handle_control_command(&control_sock, &mut services)?,
                DBUS     => handle_dbus_message(&dbus_conn, &services)?,
                token    => handle_pidfd_exit(token, &mut services, &mut poll)?,
            }
        }
    }
}
```

Each spawned service gets a pidfd registered with `mio`:

```rust
fn spawn_service(svc: &ServiceConfig, poll: &mut Poll) -> Result<ServiceHandle> {
    // clone3 returns both child PID and a pidfd
    let (pid, pidfd) = clone3_with_pidfd(|| {
        // Inside child: set up sandbox before exec
        enter_cgroup(&svc.cgroup_path)?;
        apply_seccomp_filter(&svc.seccomp_profile)?;
        set_selinux_context(&svc.selinux_type)?;
        drop_capabilities(&svc.ambient_caps)?;
        if let Some(ns) = &svc.namespaces {
            enter_namespaces(ns)?;
        }
        exec_service(svc)
    })?;

    // Register pidfd with epoll — we get notified exactly when
    // this specific process exits, no PID-reuse races possible
    let token = Token(pidfd.as_raw_fd() as usize);
    poll.registry().register(&pidfd, token, Interest::READABLE)?;

    Ok(ServiceHandle { pid, pidfd, token, config: svc.clone() })
}
```

When the pidfd becomes readable, the process has exited. No signal handler, no race condition, no scanning — just an epoll event on the exact file descriptor.

</details>

<details>
<summary>Compatibility strategy: minimal code, maximum coverage</summary>

The key insight is that most systemd "compatibility" falls into two categories:

**Category 1 — Standalone tools that work without systemd as PID 1.** These are separate binaries that read config files and do one thing. NexisOS invokes them directly during boot rather than reimplementing them:

| Tool | Purpose | Invoked during |
| :--- | :------ | :------------- |
| `systemd-tmpfiles` | Creates `/tmp`, `/run`, required directories | Early boot |
| `systemd-sysusers` | Creates system users and groups from declarative configs | Early boot |
| `systemd-sysctl` | Applies kernel parameters from `/etc/sysctl.d/` | Early boot |
| `systemd-modules-load` | Loads kernel modules from `/etc/modules-load.d/` | Early boot |
| `eudev` (or `systemd-udevd`) | Device management and hotplug events | Early boot, persistent |

These tools are maintained by the systemd project, are well-tested, and have no dependency on systemd being PID 1. Using them avoids reimplementing complex, well-solved problems.

**Category 2 — Protocol interfaces that applications talk to at runtime.** These must be implemented natively in the init:

| Interface | What expects it | Implementation |
| :-------- | :-------------- | :------------- |
| **`sd_notify`** protocol | Services using `Type=notify` (PostgreSQL, nginx, etc.) | Unix datagram socket at `/run/nexis/notify`; parses `READY=1`, `STATUS=`, `MAINPID=`, `WATCHDOG=1` messages. ~200 lines. |
| **Socket activation** (`LISTEN_FDS`) | Services expecting pre-opened sockets (systemd socket units) | Init opens sockets, passes FDs to child via `LISTEN_FDS` + `LISTEN_PID` environment variables. ~300 lines. |
| **`org.freedesktop.systemd1`** D-Bus API | `systemctl`, desktop environments, Cockpit, monitoring tools | Subset implementation via `zbus` (pure Rust D-Bus library). Exposes: `ListUnits`, `StartUnit`, `StopUnit`, `RestartUnit`, `GetUnit`, property queries. ~1000 lines for the 80% case. |
| **`/run/systemd/` state layout** | Tools that read PID files or check system state from filesystem | Symlinked: `/run/systemd/notify` → `/run/nexis/notify`, state files written to expected paths. ~50 lines. |

The `org.freedesktop.systemd1` D-Bus interface is the only substantial piece. `zbus` handles all D-Bus protocol details (authentication, marshaling, bus registration), so the init only needs to implement the method handlers — not the transport.

</details>

<details>
<summary>Unit file support</summary>

`nexis-init` reads both TOML service declarations (native) and systemd unit files (compatibility). Both compile to the same internal representation:

```toml
# Native TOML declaration — /etc/nexis/services.toml
[services.nginx]
exec = "/usr/sbin/nginx"
type = "notify"                # forking | simple | notify | oneshot
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

Supported systemd unit directives (covers the vast majority of real-world unit files):

| Section | Supported directives |
| :------ | :------------------- |
| `[Unit]` | `Description`, `After`, `Before`, `Requires`, `Wants`, `Conflicts`, `ConditionPathExists`, `ConditionFileIsExecutable` |
| `[Service]` | `Type` (simple, forking, notify, oneshot), `ExecStart`, `ExecStartPre`, `ExecStartPost`, `ExecStop`, `ExecReload`, `Restart`, `RestartSec`, `TimeoutStartSec`, `TimeoutStopSec`, `WatchdogSec`, `User`, `Group`, `WorkingDirectory`, `Environment`, `EnvironmentFile`, `MemoryMax`, `CPUWeight`, `CPUQuota`, `IOWeight`, `ProtectSystem`, `ProtectHome`, `PrivateTmp`, `NoNewPrivileges`, `AmbientCapabilities`, `CapabilityBoundingSet`, `SeccompFilter` |
| `[Install]` | `WantedBy`, `RequiredBy`, `Alias` |
| `[Timer]` | `OnCalendar`, `OnBootSec`, `OnUnitActiveSec`, `Persistent`, `Unit` |
| `[Socket]` | `ListenStream`, `ListenDatagram`, `Accept`, `Service` |

Unsupported directives are logged as warnings, not errors — the unit still loads with best-effort behavior.

</details>

<details>
<summary>Service lifecycle</summary>

```
                     ┌──────────┐
                     │ inactive │
                     └────┬─────┘
                          │ start requested
                          ▼
              ┌───────────────────────┐
              │  resolve dependencies │
              │  (check Requires=,    │
              │   start After= units) │
              └───────────┬───────────┘
                          │ deps satisfied
                          ▼
                   ┌─────────────┐
                   │ activating  │─── ExecStartPre= commands
                   └──────┬──────┘
                          │
                          ▼
              ┌───────────────────────┐
              │      spawning         │
              │  clone3(CLONE_PIDFD)  │
              │  → cgroup placement   │
              │  → namespace entry    │
              │  → seccomp load       │
              │  → SELinux transition  │
              │  → capability drop    │
              │  → exec               │
              └───────────┬───────────┘
                          │
                ┌─────────┴──────────┐
                │                    │
          Type=simple          Type=notify
          (immediate)          (waits for READY=1
                │               on notify socket)
                │                    │
                ▼                    ▼
             ┌────────┐         ┌────────┐
             │ active │         │ active │
             └───┬────┘         └───┬────┘
                 │                  │
          pidfd readable     pidfd readable
          (process exited)   (process exited)
                 │                  │
                 ▼                  ▼
          ┌─────────────┐   ┌─────────────┐
          │ deactivating│   │ deactivating│
          │ ExecStop=   │   │ ExecStop=   │
          └──────┬──────┘   └──────┬──────┘
                 │                  │
                 ▼                  ▼
          ┌────────────┐    ┌────────────┐
          │  evaluate  │    │  evaluate  │
          │  Restart=  │    │  Restart=  │
          └────┬───────┘    └────┬───────┘
               │                 │
        ┌──────┴──────┐   ┌─────┴───────┐
        ▼             ▼   ▼             ▼
   ┌────────┐   ┌──────────┐      ┌──────────┐
   │inactive│   │restarting│      │  failed   │
   └────────┘   │(RestartSec│     └──────────┘
                │ timer)    │
                └─────┬─────┘
                      │ timer fires
                      ▼
                (back to activating)
```

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
<summary>Rust crate dependencies</summary>

The init is deliberately conservative about dependencies to keep the binary small, auditable, and predictable:

| Crate | Purpose | Why this one |
| :---- | :------ | :----------- |
| `mio` | epoll event loop | Thin wrapper, no runtime, no allocations in the poll path |
| `zbus` (sync mode) | D-Bus systemd1 API | Pure Rust, no `libdbus` C dependency; sync mode avoids pulling in an async executor |
| `nix` (Rust crate) | Linux syscalls: `clone3`, `pidfd_open`, `signalfd`, `timerfd`, cgroups, namespaces, `mount` | Thin safe wrappers, widely audited |
| `toml` | Parse TOML service declarations | Standard Rust TOML parser |
| `serde` | Deserialize unit configs | Required by `toml` anyway |
| `log` + `syslog` | Logging to kernel ring buffer and syslog | No-std-compatible logger |

**Not used:** `tokio` (runtime overhead), `hyper`/`reqwest` (no HTTP needed in PID 1), `clap` (PID 1 takes no CLI arguments — `nexisctl` is a separate binary).

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

The `systemctl` shim is a symlink or thin wrapper that translates systemctl arguments to nexisctl calls. Applications and scripts using `systemctl` work without modification.

</details>

<details>
<summary>Performance comparison (design targets)</summary>

| Metric | systemd | dinit | s6-rc | nexis-init (target) |
| :----- | :------ | :---- | :---- | :------------------ |
| **PID 1 RSS** | ~12 MB | ~1 MB | ~0.5 MB | <2 MB (Rust binary + mmap'd state) |
| **Boot to multi-user** | ~1.5–3s | ~0.8–1.5s | ~0.6–1.2s | <1s (parallel startup, pidfd — no signal overhead, socket activation) |
| **Process tracking** | cgroups + SIGCHLD | SIGCHLD + waitpid | SIGCHLD + `s6-supervise` per service | pidfd via epoll — O(1) per exit, zero-cost poll, no signal handler overhead |
| **Service restart detection** | cgroup notification + PID tracking | Poll-based supervision | Per-service supervisor process | pidfd becomes readable — single epoll event, no polling, no extra processes |
| **Memory per service** | ~100–200 KB (cgroup accounting + journal) | ~30 KB | ~50 KB (dedicated supervisor process) | ~8 KB (pidfd + state struct, no per-service process) |
| **Dependency resolution** | Generator + unit graph (evaluated at runtime) | Ahead-of-time ordered list | Compiled dependency database | In-memory DAG, topological sort at load time, cached between reloads |
| **D-Bus API** | Full `org.freedesktop.systemd1` | None | None | Subset (ListUnits, Start/Stop/Restart, GetUnit, properties) — covers 80%+ of real-world callers |
| **systemctl compat** | Native | No | No | Shim binary — translates CLI to nexisctl |
| **sd_notify compat** | Native | Partial (via `dinitctl`) | No | Full protocol (READY, STATUS, MAINPID, WATCHDOG, FDSTORE) |
| **Socket activation** | Native | Partial | Via `s6-ipcserver` | Full LISTEN_FDS protocol |
| **Watchdog** | `WatchdogSec=` in units | No | Via `s6-supervise` | timerfd per service — fires into same epoll loop |

> **Note:** These are design targets. Actual benchmarks will be published once the implementation stabilizes.

</details>

<details>
<summary>Init security model</summary>

Every service spawned by `nexis-init` runs in a security sandbox by default:

| Layer | Mechanism | Default |
| :---- | :-------- | :------ |
| **Process isolation** | Linux namespaces (mount, PID, IPC, UTS, network — per service config) | mount + IPC namespaces enabled |
| **Capability restriction** | Ambient capabilities + bounding set | All capabilities dropped except those explicitly declared |
| **Syscall filtering** | seccomp-BPF profiles loaded before exec | Default restrictive profile (denies `kexec_load`, `reboot`, `mount`, etc.) |
| **MAC** | SELinux context transition at exec time | Enforced — service type declared in TOML or unit file |
| **Resource limits** | cgroups v2 (memory, CPU, IO, PIDs) | Default PID limit (4096), no memory/CPU limit unless declared |
| **Filesystem** | Read-only root (erofs), writable paths declared per-service | `/` is read-only; services get writable access only to declared paths |

Services can relax restrictions explicitly in their config, but the default is restrictive. This inverts the typical Linux init model where services run with full privileges unless explicitly sandboxed.

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
BuildHash = blake3(
    source inputs +
    build script +
    dependency BuildHashes +
    build environment metadata
)

InterfaceHash = blake3(
    exported headers/symbols +
    ABI-relevant outputs +
    public API surface
)
```

**BuildHash** determines whether a package itself needs to be rebuilt. **InterfaceHash** determines whether *dependents* need to be rebuilt. If a dependency's BuildHash changes but its InterfaceHash stays the same (e.g., an internal refactor, comment change, or build script tweak that doesn't affect outputs), downstream packages are not rebuilt.

This two-level model is where NexisOS achieves its largest rebuild reduction compared to NixOS, where any change to a dependency's derivation hash forces all dependents to rebuild regardless of whether the output ABI changed.

Additional properties:

- Binary substitution works across machines even when the local store layout differs.
- Rebuild scope is proportional to what actually changed at the interface boundary, not to graph depth.
- An optional `--check` mode builds a package twice and compares outputs to detect non-determinism, compensating for the narrower hash scope.

</details>

<details>
<summary>Content-Addressed Store (CAS)</summary>

NexisOS uses a CAS for **storage and identity** but avoids using it as the runtime filesystem directly. The store is the source of truth; activation compiles packages into an immutable erofs image that is mounted as the root filesystem.

**Why not pure mounts (no CAS)?** Without content addressing you lose: deduplication (identical files across packages stored once), reproducibility verification (hash-based identity), binary substitution (fetch pre-built packages by hash), and atomic rollback (generations reference immutable objects).

**Why not pure CAS (like NixOS)?** Exposing hashed store paths to processes breaks FHS assumptions, prevents SELinux labeling, and requires symlink forests that are slow to create and maintain.

**The design:** Store in CAS, compile to erofs, mount.

```
/store/
    /objects/<blake3>            # immutable file objects (deduplicated)
    /packages/<PackageID>/       # logical package — FHS-structured tree
        manifest.toml
        files.list
        dependencies.list
```

- **Objects** store immutable file data addressed by BLAKE3 content hash. Identical files across packages are stored once.
- **Packages** reference sets of CAS objects and declare dependency relationships. Each package's tree is laid out in FHS structure (`usr/bin/`, `usr/lib/`, etc.) so it can be composed directly into a generation image.
- **Deduplication** happens at insert time — the store never needs a global optimization pass.
- **Metadata DB** LMDB (Lightning Memory-Mapped Database) with separate named databases for content index (`ContentID → object path`), package metadata (`PackageID → content list, interface hash`), generation history, and build coordination. LMDB is chosen over RocksDB because NexisOS's workload is read-dominant: every `nexis shell` invocation, dependency resolution, and generation activation reads the package index, while writes only occur during package install. LMDB's B+ tree is memory-mapped — reads are a pointer dereference into mmap'd pages with zero serialization overhead and zero reader locking. LMDB transactions are atomic via copy-on-write pages: a failed write never corrupts existing data, eliminating the need for a separate write-ahead log at the database layer.
- **Crash safety** LMDB's copy-on-write page semantics provide atomic transactions without a WAL. A package install opens a write transaction, inserts CAS object references and the package manifest, and commits. If the process crashes mid-transaction, the old B+ tree pages are still intact, no orphaned objects, no recovery pass needed.
- **Garbage collection** walks committed package manifests to build the live reference set, then removes unreferenced objects.

</details>

<details>
<summary>Generation Projection & Rollback</summary>

```
/generations/<id>/rootfs.erofs   # compressed, immutable FHS image for this generation
/system → /generations/<id>/     # atomic symlink to active generation (erofs mounted here)
```

- Each generation is a single **erofs image** compiled from the CAS at build time via `mkfs.erofs`.
- The image contains the complete FHS tree (`/usr/bin`, `/usr/lib`, `/etc` base layer, etc.) with LZ4 compression.
- SELinux labels are baked into the erofs image during compilation xattrs are set on the source tree before `mkfs.erofs` runs.
- Symlink switch enables **O(1) rollbacks** switching generations remounts a different erofs image.
- erofs is read-only by design, no need for `upperdir=none` hacks or mount flags to enforce immutability.
- Generation images are typically 500MB–2GB compressed (depending on installed packages), and build in 1–5 seconds.

</details>

<details>
<summary>Filesystem Projection</summary>

NexisOS uses a **two-tier projection model**: erofs for system generations, OverlayFS for ephemeral environments.

#### Tier 1 erofs (system activation)

At generation build time, the package manager composes all declared packages into a single FHS directory tree, applies SELinux labels, then compiles it into a compressed erofs image:

```
# Build phase (happens once per generation):
1. Resolve declared packages from CAS → assemble FHS tree in /tmp/gen-build/
2. Apply SELinux labels to the assembled tree
3. mkfs.erofs -zlz4 /generations/42/rootfs.erofs /tmp/gen-build/

# Activation (happens on boot or `nexis switch`):
mount -t erofs /generations/42/rootfs.erofs /system
```

Why erofs over OverlayFS for system activation:

- **No layer limits:** OverlayFS caps at ~128 lower layers per kernel version. A desktop system with hundreds of packages would require pre-merging layers to stay under the limit. erofs has no such constraint — the entire system is one image.
- **Immutable by design:** erofs is a read-only filesystem at the kernel level. No need for OverlayFS `upperdir=none` or mount flags to prevent writes.
- **LZ4 compression:** the generation image is smaller than the sum of its packages on disk. Block-aligned decompression means read performance is near-native.
- **Single mount:** activation is one `mount` call regardless of how many packages are installed.
- **SELinux xattr support:** erofs supports extended attributes natively; labels baked in at build time are visible to the running system.
- **ext4 compatible:** erofs images are regular files on the backing ext4 filesystem. No btrfs, XFS, or special kernel features required beyond `CONFIG_EROFS_FS` (mainline since Linux 5.4).

#### Tier 2 — OverlayFS (ephemeral environments / `nexis shell`)

For `nexis shell`, where sub-100ms activation matters and only a few packages are being added, OverlayFS is used within a mount namespace:

```
# nexis shell adds packages on top of the erofs base:
unshare --mount bash -c '
    mount -t overlay overlay -o \
        lowerdir=/store/packages/<python312>:/store/packages/<nodejs20>:/system \
        /system
    exec $SHELL
'
```

This stays well within OverlayFS layer limits (the erofs base counts as one layer, plus a handful of shell-requested packages) and avoids the `mkfs.erofs` build step entirely.

#### Tier summary

| Use case | Mechanism | Latency | Layer limit concern |
|---|---|---|---|
| System generation | erofs image | 1–5s build, instant mount | None — single image |
| `nexis shell` | OverlayFS over erofs base | <100ms | No — base is 1 layer + few packages |
| Rollback | Remount different erofs image | <1ms | None |

</details>

<details>
<summary>Ephemeral Environments <code>nexis shell</code></summary>

NexisOS replaces `nix-shell` with `nexis shell`, an ephemeral development environment system built on **Linux mount namespaces** and **OverlayFS over the erofs system base** instead of building new derivations or copying files.

#### How it works

1. **`nexis shell python312 nodejs20 gcc`** — the user requests a shell with additional packages.
2. The tool resolves packages from the CAS (fetching/building only if not already present).
3. A new **mount namespace** is created via `unshare(CLONE_NEWNS)`.
4. The active erofs system image is used as the base layer, with requested packages injected as additional OverlayFS lower layers on top.
5. Environment variables (`PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH`, etc.) are computed from the package manifests and exported into the new shell.
6. The user gets a shell with all requested tools available. **No files are copied. No symlink forests. No new derivations built. No erofs rebuild.**

```
# Activation is a mount operation, not a build:
unshare --mount bash -c '
    mount -t overlay overlay -o \
        lowerdir=/store/packages/<python312>:/store/packages/<nodejs20>:/system \
        /system
    exec $SHELL
'
# The erofs base (/system) counts as 1 layer — adding a few packages
# stays far under OverlayFS's ~128 layer limit.
```

#### Why this is faster than nix-shell

| | nix-shell | nexis shell |
|---|---|---|
| **Cold start** (packages cached) | Evaluates Nix expression → builds shell derivation → creates gcroot → enters | Resolves manifests → `unshare` + `mount` → enters |
| **Typical latency** | 2–15s even with cached packages | <100ms for cached packages |
| **Mechanism** | New derivation with merged environment | Mount namespace overlay injection |
| **Cleanup** | Gcroot removal + eventual GC | Namespace destroyed on shell exit (zero cleanup) |
| **Composability** | Nested shells are slow and fragile | Nested namespaces stack cleanly |

#### `nexis shell` TOML lockfiles

For reproducible project environments (like `shell.nix` or `flake.nix` devShells):

```toml
# nexis-shell.toml — checked into version control
[shell]
packages = ["python@3.12.4", "nodejs@20.11", "postgresql@16"]
env.DATABASE_URL = "postgresql://localhost/devdb"

[shell.hooks]
enter = "echo 'Dev environment ready'"
exit = "pg_ctl stop -D $PGDATA 2>/dev/null || true"

[shell.services]
# Ephemeral services that run only inside the shell
postgresql = { exec = "pg_ctl start -D $PGDATA", stop = "pg_ctl stop -D $PGDATA" }
```

Run `nexis shell` in a directory with this file and the environment is reconstructed identically on any NexisOS machine.

#### Container export

`nexis shell --export-oci` produces an OCI container image from the shell environment, bridging development and deployment.

</details>

<details>
<summary>Distribution & Updates</summary>

- **HTTP/3 (QUIC)** for low-latency binary fetches.
- **BitTorrent / IPFS-style replication** for community mirrors and peer distribution.
- **Delta updates** only changed CAS objects are transferred, not full packages.

</details>

---

## 🔒 Filesystem Access Policy

<details>
<summary>View directory mutability table</summary>

| Directory | Mutable | Notes |
| :-------- | :------ | :---- |
| `/store` | ❌ | Immutable content-addressed store |
| `/generations/<id>/rootfs.erofs` | ❌ | Compressed erofs image, SELinux labeled at build time |
| `/system` | ❌ | Symlink to active generation |
| `/etc` | Partial | Three-layer merge: (1) generation base from TOML-declared configs, (2) writable overlay for runtime state (`resolv.conf`, `machine-id`), (3) `/etc/local/` for admin overrides. The writable layer resets on generation switch unless persisted in TOML. |
| `/var/lib` | Partial | Application state; persistence declared per-service in TOML |
| `/var/log`, `/run`, `/tmp` | ✅ | Runtime and temporary data |
| `/home/<user>/.config`, `.local/share` | ❌ | Declared in TOML (user environment) |
| `/home/<user>/Documents`, `Downloads`, etc. | ✅ | User-controlled directories |

</details>

---

## 📁 Profiles & Configuration Directory

NexisOS separates **configuration source** (what you write and version-control) from **runtime state** (what the tooling manages). Profiles are composable configuration layers, the unit of reuse that works identically on a single machine and across a fleet.

<details>
<summary>Configuration source, profile structure, and runtime state</summary>

### Configuration source — `/etc/nexis/`

```
/etc/nexis/                              # version-controlled (git init here)
    system.toml                          # base: kernel, bootloader, locale, timezone, filesystem
    hardware.toml                        # auto-generated by installer, editable (GPU, drivers, firmware)
    packages.toml                        # system-wide package declarations
    services.toml                        # service declarations and overrides
    users.toml                           # user accounts, groups, home environment declarations

    profiles/                            # named, composable configuration layers
        workstation.toml                 # desktop environment, GUI, audio, Bluetooth
        server.toml                      # headless, hardened defaults, monitoring agents
        gaming.toml                      # GPU drivers, Steam, Proton, gamemode
        development.toml                 # compilers, LSPs, debug tools, container runtimes

    modules/                             # reusable config fragments imported by profiles
        nginx.toml                       # nginx service + package + SELinux policy
        postgresql.toml                  # postgresql service + tuning presets
        monitoring.toml                  # prometheus + node-exporter + alertmanager
```

A machine's **effective configuration** is computed by merging:

```
system.toml + hardware.toml + packages.toml + services.toml + users.toml
    + profiles/<active profiles merged in declaration order>
        + modules/<any modules imported by active profiles>
```

Later layers override earlier ones. Conflicts are resolved by declaration order with explicit errors for ambiguous merges (e.g., two profiles setting the same service option to different values without one declaring `priority`).

### Profile TOML structure

```toml
# /etc/nexis/profiles/development.toml

[profile]
name = "development"
description = "Compilers, language servers, and debugging tools"
imports = ["modules/monitoring.toml"]   # pull in reusable fragments

[profile.packages]
add = [
    "gcc@14", "clang@18", "rust-analyzer", "gdb", "lldb",
    "nodejs@22", "python@3.12", "go@1.22",
    "docker", "podman",
]

[profile.services]
docker = { enable = true }
podman = { enable = true, rootless = true }

[profile.env]
# Environment variables set system-wide when this profile is active
EDITOR = "nvim"
RUST_BACKTRACE = "1"

[profile.shell]
# Default nexis shell packages when this profile is active
packages = ["rust-analyzer", "clang-tools"]
```

```toml
# /etc/nexis/profiles/gaming.toml

[profile]
name = "gaming"
description = "GPU drivers, Steam, and game performance tools"

[profile.packages]
add = ["mesa-git", "vulkan-tools", "steam", "proton-ge", "gamemode", "mangohud"]

[profile.services]
gamemode = { enable = true }

[profile.kernel]
# Profile-specific kernel parameters
parameters = ["amd_pstate=active", "split_lock_detect=off"]
```

### Activating profiles

```bash
# Single machine — apply profiles and rebuild
nexis profile enable development gaming
nexis rebuild                              # builds new erofs generation with merged config

# List active profiles
nexis profile list
# ● development    Compilers, language servers, and debugging tools
# ● gaming         GPU drivers, Steam, and game performance tools
# ○ server         (inactive)
# ○ workstation    (inactive)

# Disable a profile and rebuild
nexis profile disable gaming
nexis rebuild

# Preview what a profile change would do without building
nexis profile diff +server -gaming         # show package/service diff
```

### Runtime state — `/nexis/`

```
/nexis/                                    # managed by nexis tooling (not user-edited)
    db/
        metadata.lmdb                      # package metadata, content index, build coordination
        generations.lmdb                   # generation history and profile snapshots

    generations/
        42/
            rootfs.erofs                   # compressed FHS image
            manifest.toml                  # snapshot: which profiles, packages, services
            config-hash                    # blake3 of the effective merged config
        43/
            rootfs.erofs
            manifest.toml
            config-hash

    active -> 43                           # symlink to current generation ID (atomic switch)

    cache/
        build/                             # intermediate build artifacts
        fetch/                             # downloaded source tarballs, binary substitutes
```

Each generation's `manifest.toml` records the exact profiles and package versions that produced it:

```toml
# /nexis/generations/43/manifest.toml (auto-generated, read-only)
[generation]
id = 43
timestamp = "2026-03-15T14:22:07Z"
config_hash = "blake3:a7c3f..."

[generation.profiles]
active = ["development", "gaming"]

[generation.packages]
# full resolved package list with exact versions and store paths
coreutils = { version = "9.5", store = "/store/packages/coreutils-9.5-ab3f..." }
gcc = { version = "14.1", store = "/store/packages/gcc-14.1-7de2..." }
# ... (hundreds of entries)
```

### How profiles connect to fleet

Profiles are the bridge between single-machine configuration and fleet-wide management. Fleet **roles** resolve to **profiles** — they are the same abstraction at different scales.

```toml
# nexis-fleet.toml — roles reference the same profiles used locally

[roles.base]
profiles = ["server"]                          # every host gets the server profile
packages = ["monitoring-agent"]                # role can add packages on top

[roles.webserver]
inherits = ["base"]
profiles = ["server"]                          # inherited from base, listed for clarity
packages = ["nginx", "certbot"]

[roles.database]
inherits = ["base"]
profiles = ["server"]
packages = ["postgresql@16"]

[hosts.web-01]
roles = ["webserver"]
address = "10.0.1.10"
profiles = ["webserver-tuned"]                 # host-specific profile override/addition

[hosts.dev-workstation]
roles = []                                     # no fleet role — standalone machine
profiles = ["workstation", "development"]      # uses the same profile files
address = "10.0.2.50"
```

The fleet resolver computes each host's effective config the same way a standalone machine does — merge base TOML + active profiles + role packages + host overrides — then diffs against the host's current generation. If a host's effective config hash matches its running generation, it is skipped entirely.

```
Fleet deployment flow:

  For each host:
    1. Merge: system.toml + profiles from role + host overrides → effective config
    2. Hash effective config → compare to host's current generation config_hash
    3. If unchanged → skip (no rebuild, no sync, no downtime)
    4. If changed  → diff packages → build only new/changed packages
                   → compile erofs image → delta-sync to host → activate
```

This means a fleet-wide "deploy" where only one profile changed on a subset of hosts will skip every unaffected machine, rebuild only the changed packages, and produce new erofs images only for the hosts that need them.

</details>

---

## 🌐 Fleet Orchestration `nexis fleet`

NexisOS includes a built-in fleet management system for deploying and managing configurations across many machines from a single declaration. This replaces tools like NixOps, colmena, and deploy-rs with a first-class, integrated solution.

<details>
<summary>Fleet architecture, declaration, deployment strategies, and commands</summary>

### Architecture

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

### Fleet declaration

```toml
# nexis-fleet.toml

[fleet]
build_host = "builder.internal"       # optional dedicated builder
strategy = "rolling"                   # rolling | canary | blue-green | parallel
config_repo = "git@github.com:org/infra.git"  # shared /etc/nexis/ across fleet

[fleet.defaults]
timezone = "UTC"
locale = "en_US.UTF-8"

# Roles compose — a host can have multiple roles
# Roles reference profiles from /etc/nexis/profiles/
[roles.base]
profiles = ["server"]
packages = ["openssh", "monitoring-agent"]
services = ["sshd", "node-exporter"]

[roles.webserver]
inherits = ["base"]
packages = ["nginx", "certbot"]
services = ["nginx"]

[roles.database]
inherits = ["base"]
packages = ["postgresql@16", "pg-bouncer"]
services = ["postgresql"]
storage.persistent = ["/var/lib/postgresql"]

# Per-host — same profiles, same merge rules as standalone machines
[hosts.web-01]
roles = ["webserver"]
address = "10.0.1.10"

[hosts.web-02]
roles = ["webserver"]
address = "10.0.1.11"

[hosts.db-01]
roles = ["database"]
address = "10.0.1.20"
# Host-specific override
[hosts.db-01.postgresql]
max_connections = 200
shared_buffers = "4GB"
```

### Deployment strategies

| Strategy | Behavior |
|---|---|
| **rolling** | Activates hosts sequentially; stops on first failure; automatic rollback of failed host |
| **canary** | Deploys to a tagged subset first; waits for health checks; proceeds or aborts |
| **blue-green** | Builds full new generation across fleet; switches all hosts atomically via symlink |
| **parallel** | Deploys to all hosts simultaneously (for non-critical or idempotent updates) |

### Distributed Build Mesh

For large fleets, building everything on one machine is a bottleneck. The build mesh distributes work:

1. The fleet resolver computes the full dependency DAG across all host configurations.
2. Shared packages (e.g., `coreutils` needed by every host) are built once.
3. Build jobs are distributed across fleet members with spare capacity using a **work-stealing scheduler**.
4. Built artifacts are pushed to a shared CAS (or the fleet's binary cache) for other hosts to pull.
5. Each host receives only the CAS objects it doesn't already have (delta sync).

```
Fleet DAG:
  coreutils ←── web-01, web-02, db-01  (build once, distribute)
  nginx     ←── web-01, web-02          (build once, share)
  postgresql ←── db-01                  (build once, send to db-01)

Build assignment:
  builder.internal → coreutils, nginx
  db-01 (idle CPU) → postgresql
```

### Fleet commands

```bash
nexis fleet plan                 # show what would change per host
nexis fleet deploy               # execute deployment with configured strategy
nexis fleet deploy --hosts web-01  # deploy to specific host(s)
nexis fleet rollback web-01      # rollback a specific host
nexis fleet status               # show generation, profiles, and health per host
nexis fleet diff web-01 web-02   # compare effective configs between hosts
nexis fleet sync-profiles        # push /etc/nexis/profiles/ to all hosts from config_repo
```

</details>

---

## 📂 Repository Structure

| Repository | Purpose |
| :--------- | :------ |
| **nexisos-bootstrap** | Core distribution packages bootstrapped/downloaded by the installer |
| **nexisos-installer-iso** | Minimal TUI-based installer ISO with WiFi support |

### 📦 nexisos-bootstrap

<details>
<summary>Click to see</summary>

```text
nexisos-bootstrap/
├── nexis_common/   # Shared utilities and core logic
├── nexis_init/     # System initialization components
├── nexis_pm/       # Package manager (core logic, tests, benches)
├── nexis_scan/     # System and package scanning utilities
├── Cargo.toml      # Rust workspace definition
└── diagram1.puml   # Architecture diagram
```

</details>

### 💿 nexisos-installer-iso

<details>
<summary>Click to see</summary>

```text
nexisos-installer-iso/
├── buildroot/        # Buildroot (git submodule, ISO base system)
├── distroConfigs/    # NexisOS-specific configs and overlays
│   ├── overlay/      # Root filesystem overlay
│   ├── scripts/      # Build and install helper scripts
│   └── buildroot-packages/
├── installer/        # Rust-based TUI installer
├── createdISOs/      # Generated ISO output
├── Makefile          # Build entrypoint
└── shell.nix         # Development environment
```

### 🔧 Setup

This repository uses git submodules.
git clone --recurse-submodules https://github.com/NexisOS/nexisos-installer-iso

**Or if already cloned:**
git submodule update --init --recursive

</details>

---

## 📥 Download

ISO builds will be available on SourceForge once the first release is ready:

👉 [NexisOS on SourceForge](https://sourceforge.net/projects/nexisos/files/latest/download)

---

## 🙏 Acknowledgments

<details>
<summary>Projects that inspired or support NexisOS</summary>

- **NixOS:** Declarative system configuration and reproducible builds
- **Rust community:** Language powering the package manager and init system
- **mio** & **zbus:** Event loop and D-Bus crates enabling a minimal, correct PID 1
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

Certain jurisdictions—including some U.S. states and foreign countries—may impose age-verification, identity, or monitoring requirements related to operating systems or online platforms. NexisOS does not currently implement such mechanisms, and no widely adopted cross-architecture standard exists for doing so across the diverse systems it supports.

Users, integrators, and redistributors are solely responsible for ensuring compliance with all applicable laws in their jurisdiction before installing, distributing, or deploying this software.

## Policy and Standards

NexisOS will be publishing an open letter requesting technical clarification and
standards coordination regarding operating-system-level age assurance
requirements affecting decentralized open-source systems to respective legislators, relevant regulatory and standards bodies.
