<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

# Overview

NexisOS is a free and open-source Linux distribution focused on **transparency,
control, reproducibility, and system integrity** through declarative
configuration.

It is built around:

- A custom package manager
- A custom init system (`nexis-init`)
- TOML-based system configuration

The design is inspired by NixOS, but rethinks key areas such as filesystem
layout, SELinux compatibility, and system composition. Core components are
primarily written in **Rust** and organized as a **Cargo workspace**, with **C
used where low-level integration or direct system interfacing is more
appropriate**.

---

## Key Design Principles

- **Declarative Configuration** — System state defined in TOML
- **Reproducibility** — Builds derive strictly from declared inputs
- **Separation of Concerns** — Build, storage, and runtime are distinct layers
- **Standard Compatibility** — FHS layout and common Linux interfaces
- **Security-first Design** — SELinux and sandboxing integrated from the start

---

## Motivation

NexisOS explores an alternative approach to declarative Linux systems by addressing several practical limitations observed in existing designs.

### Comparison with NixOS

| Area              | NixOS                  | NexisOS                      |
| ----------------- | ---------------------- | ---------------------------- |
| Configuration     | Nix language (DSL)     | TOML (schema-validated)      |
| Filesystem layout | `/nix/store`           | Standard FHS projection      |
| SELinux           | Difficult to integrate | First-class design goal      |
| Init system       | systemd                | Custom (`nexis-init`)        |
| Package identity  | Single derivation hash | Build + interface separation |

### Design Philosophy

NexisOS is a system for **defining systems**, not distributing pre-built software.

Instead of maintaining a centralized package repository, NexisOS focuses on **reproducible build instructions (derivations)** that describe how software is obtained and composed into a working system.

This shifts the model from:
- *"installing packages"* → to → *"declaring system state"*
- *"trusted repositories"* → to → *"reproducible definitions"*

The goal is to keep the system transparent, auditable, and fully controlled by configuration. This also lets collaborators and maintainers focus on core design features rather than package compatibility.

---

## ⚠️ Project Status

NexisOS is in **early development and not yet in pre-alpha**.

- No official releases or ISOs are available
- Core architecture is still evolving
- Features described in this document may change

This project is currently intended for **design, experimentation, and development only**.

### Current Scope/Focus

**Implemented / in progress:**
- System architecture design
- Core components (package manager, init system)
- Configuration model

<div align="center">

| Build Status | Latest Stable Release |
|--------------|-----------------------|
| ![nexisos](https://img.shields.io/github/actions/workflow/status/NexisOS/nexisos/main.yml?label=nexisos) | ![GitHub release](https://img.shields.io/github/v/release/NexisOS/nexisos?label=latest%20stable) |

</div>

ISO builds will be available on SourceForge once the first release is ready:

[NexisOS on SourceForge](https://sourceforge.net/projects/nexisos/files/latest/download)

---

## In-Depth Design

<details>
<summary>Click to see</summary>

### System Architecture

```text
TOML Configuration
        ↓
Dependency Graph
        ↓
Package Build
        ↓
Content-Addressed Store
        ↓
Generation Image (erofs)
        ↓
System Activation
```

---

### Init System — `nexis-init`

NexisOS uses a custom init system written in Rust, designed to be minimal, deterministic, and auditable while leveraging modern Linux kernel primitives instead of large userspace abstractions.

`nexis-init` follows a primarily single-threaded, event-driven architecture built around `pidfd`, `epoll`, `signalfd`, and `timerfd` for reliable process supervision and predictable system behavior.

**Key capabilities:**
- pidfd-based process supervision
- epoll-driven event loop
- cgroups v2 integration
- Socket activation support
- `sd_notify` compatibility
- Limited `systemd` D-Bus compatibility where practical
- Namespace and seccomp sandboxing
- Deterministic service lifecycle management
- Hard timeout enforcement for shutdown reliability
- Minimal IPC and dependency complexity

**Design goals:**
- Fast and reliable boot/shutdown
- No indefinite shutdown hangs
- No shell-based service execution
- Minimal PID1 attack surface
- Strong service isolation
- Predictable and transparent behavior
- Small, maintainable codebase

**Shutdown model:**
1. Stop accepting new service activations
2. Send `SIGTERM` in reverse dependency order
3. Reap exited services through `pidfd`
4. Force kill remaining services with `SIGKILL`
5. Unmount filesystems, `sync`, and reboot/poweroff

`nexis-init` intentionally avoids indefinite waits, complex service states, and long-lived "stopping" limbo states common in larger init systems.

---

### Package Management

NexisOS separates **storage**, **build**, and **runtime** while ensuring compatibility with standard Linux tooling. It does **not provide a centralized repository of pre-defined packages** — instead, it uses **declarative build instructions (derivations)** that describe how to obtain software.

Derivations can fetch **precompiled binaries** from trusted sources or build software **from source in a reproducible environment**.

#### FHS Projection and SELinux Compatibility

Unlike NixOS, where package paths change on rebuild, NexisOS is designed so that **runtime paths remain stable and compatible with tools like SELinux**.

This is achieved by:

- Keeping the **content-addressed store internal** (`/store`)
- Projecting packages into a **stable FHS layout** (`/usr/bin`, `/usr/lib`, etc.) during generation build
- Allowing SELinux and other path-based tools to apply standard policies without special handling

NexisOS avoids relying on heavy use of `rpath` for runtime resolution. Instead, it favors standard dynamic linker behavior, FHS-consistent library locations, and controlled environment composition at generation time.

#### Store, Generations, and Package Identity

**Content-Addressed Store**
- Immutable storage of build outputs
- Deduplicated by content hash

**Generations**
- System states built as immutable images
- Activated atomically
- Instant rollback via generation switching

**Package Identity**
- Build identity (internal changes)
- Interface identity (external compatibility)

---

### Filesystem Model

| Path      | Purpose                   |
| --------- | ------------------------- |
| `/store`  | Immutable package storage |
| `/system` | Active system generation  |
| `/etc`    | Configuration             |
| `/var`    | Application state         |
| `/home`   | User data                 |

---

### Security Model

Security is integrated into the system design rather than layered on top.
Unlike most Linux distributions that require manual setup and tuning of
security tools, the system is hardened by default and actively prevents threats
without requiring user intervention.

A unified security dashboard provides a clear, single view of system status
simple enough for beginners, while still offering the depth and visibility
needed for servers and large-scale deployments.

Durring install the TUI will ask user general questions which impacts what settings are appropriate for their use case.

| Layer                            | Components                                                                | Purpose                                                             |
| -------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **Kernel Hardening**             | KASLR, module signing, lockdown mode, hardened usercopy, sysctl hardening | Reduce kernel attack surface and prevent privilege escalation       |
| **Policy Enforcement**           | SELinux, seccomp, cgroups, namespaces, capabilities                       | MAC, sandboxing, least privilege execution                          |
| **Network Defense**              | nftables, CrowdSec, fail2ban                                              | Firewalling, IP reputation blocking, brute-force prevention         |
| **Network IPS**                  | Suricata                                                                  | Deep packet inspection and inline intrusion prevention              |
| **Host Runtime Protection**      | Falco, Tetragon                                                           | Real-time syscall/process detection and response                    |
| **Monitoring & SIEM**            | Wazuh, osquery                                                            | Central dashboard, correlation, system state visibility             |
| **Malware Scanning (Evaluated)** | ClamAV, LMD, planned custom scanner                                       | On-demand file scanning; exploring improved Linux-focused detection |

---

### Configuration

System configuration is defined in `/etc/nexis/` using TOML.

```
system.toml
packages.toml
services.toml
users.toml
profiles/
modules/
```

---

### Ephemeral Environments

NexisOS will support ephemeral shell environments using a mechanism similar to `nix-shell`.

</details>

---

## Community & Support

For now, use GitHub Issues for discussion and feedback.

**Organization Resources:**
- [Contributing Guidelines](https://github.com/NexisOS/.github/blob/main/CONTRIBUTING.md)
- [Code of Conduct](https://github.com/NexisOS/.github/blob/main/CODE_OF_CONDUCT.md)
- [Security Policy](https://github.com/NexisOS/.github/blob/main/SECURITY.md)
- [Governance](https://github.com/NexisOS/.github/blob/main/GOVERNANCE.md)
- [Pull Request Template](https://github.com/NexisOS/.github/blob/main/PULL_REQUEST_TEMPLATE.md)

---

## Contact

**Email:** __kyle.gortych.dev@gmail.com__ — Please use GitHub Issues for general inquiries. As the sole maintainer, response times may vary.

---

## License

NexisOS is licensed under the **GNU GPL v3.0**. This software is provided **"as is"**, without warranty of any kind. The project does not collect user data, does not provide hosted services, and is independently developed.

---

## AI usage policy

NexisOS is open to the use of AI-assisted tools in development, research, and documentation. However, all contributions—whether AI-assisted or not—must meet the same standards of quality, correctness, and security.

Requirements for AI-assisted contributions:

- **Human review is mandatory** before submission
- Contributors are responsible for understanding all code they submit
- Generated code must be verified for correctness, security, and licensing concerns

AI tools are treated as productivity aids, not authoritative sources.

For contribution standards and expectations, see the Contributing Guidelines in the Community & Support section.

---

## Legal & Policy Notes

NexisOS is an independent project and is not affiliated with any organizations, including the Linux Foundation or the NixOS Foundation. All trademarks, logos, and project names referenced belong to their respective owners.

This project aims to operate within applicable legal frameworks within the U.S. and respective . However, NexisOS does not include jurisdiction-specific compliance mechanisms by default.

**Position on Legislation**

The maintainer has concerns regarding legislation that introduces unclear, invasive, or technically impractical requirements for operating systems that are free and open-source and by nature decentralized, including proposals such as U.S. H.R. 8250. While these laws reduce the amount of times one submits ID for age verifivation and thuse improves convenience and reduces attack surface, it comes at the cost of user anonymity. Finding a balnce between privacy and public saftey has always been the .

It is also concerning the adoption of said actions by Systemd and a fear is forced usage of said init system due to .

Such proposals may raise issues around user privacy, system security, technical feasibility, and compatibility with open-source development models.

Any opinions expressed are those of the individual maintainer and do not impact the technical scope or functionality of the project.

NexisOS focuses on building transparent, user-controlled systems. It does not implement speculative or undefined compliance features unless clearly required and technically well-defined.
