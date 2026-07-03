<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">

---

**A declarative Linux distribution via NixOS-style reproducibility with a standard FHS layout, first-class SELinux, and TOML configuration.**

| Build Status | Latest Stable Release |
|--------------|-----------------------|
| ![nexisos](https://img.shields.io/github/actions/workflow/status/NexisOS/nexisos/main.yml?label=nexisos) | ![GitHub release](https://img.shields.io/github/v/release/NexisOS/nexisos?label=latest%20stable) |

</div>

NexisOS defines the entire system from declarative **TOML** configuration: packages, services, and users are built into immutable, atomically-activated generations with instant rollback. Core components are written in **Rust**.

Unlike NixOS, NexisOS projects packages into a **standard FHS layout**, so SELinux and other path-based tooling work **out of the box** no patched policies, no special handling.

---

## Why NexisOS

| Area | NixOS | NexisOS |
|---|---|---|
| Configuration | Nix language (DSL) | TOML (schema-validated) |
| Filesystem layout | `/nix/store` symlink forest | Standard FHS projection |
| SELinux | Difficult to integrate | Works out of the box |
| Store optimization | Post-hoc tree traversal (`nix-store --optimise`) | Deduplicated at write time — no optimise pass |
| Init system | systemd | `nexis-init` (Rust, minimal PID 1) |
| Package identity | Single derivation hash | Build identity + interface identity |

**Design principles:** declarative configuration · reproducible builds · security hardened by default · standard Linux compatibility · clean separation of build, storage, and runtime.

**Philosophy:** NexisOS is a system for *defining systems*, not distributing pre-built software. There is no centralized package repository — reproducible **derivations** describe how software is fetched (trusted binaries) or built from source. *"Installing packages"* becomes *"declaring system state."*

---

## ⚠️ Project Status

**Early development pre-alpha.** No releases or ISOs yet. Architecture is evolving; anything described here may change. Currently suitable for design discussion and development only.

In progress: system architecture, package manager, `nexis-init`, configuration model, TUI installer. ISOs will be published on [SourceForge](https://sourceforge.net/projects/nexisos/files/latest/download) once the first release is ready.

---

## Architecture

```text
TOML Configuration → Dependency Graph → Package Build
        → Content-Addressed Store → Generation Image (erofs) → Activation
```

- **Store (`/store`)** — immutable build outputs, content-hashed and **deduplicated on write**. Because identity is established at ingest, there is no periodic hardlink-scanning pass over the store.
- **Generations** — complete system states packed as flat, read-only **erofs images**. No deep symlink trees to resolve at runtime; activation is atomic and rollback is instant.
- **FHS projection** — packages are projected into `/usr/bin`, `/usr/lib`, etc. at generation build time. Runtime paths stay stable across rebuilds, so SELinux policies, dynamic linking, and standard tooling behave normally — no heavy `rpath` reliance.

| Path | Purpose |
|---|---|
| `/store` | Immutable package storage |
| `/system` | Active generation |
| `/etc` | Configuration (`/etc/nexis/*.toml`) |
| `/var` | Application state |
| `/home` | User data |

<details>
<summary><b>Init system — <code>nexis-init</code></b></summary>

A minimal, deterministic PID 1 in Rust, built on modern kernel primitives instead of large userspace abstractions: `pidfd`-based supervision, an `epoll`-driven event loop, `signalfd`/`timerfd`, cgroups v2, socket activation, `sd_notify` compatibility, and namespace/seccomp sandboxing.

Goals: fast, reliable boot and shutdown; no shell-based service execution; no indefinite shutdown hangs; small PID 1 attack surface.

Shutdown: stop new activations → `SIGTERM` in reverse dependency order → reap via `pidfd` → `SIGKILL` stragglers on hard timeout → unmount, `sync`, poweroff. No "stopping" limbo states.

</details>

<details>
<summary><b>Security model</b></summary>

Hardened by default rather than layered on afterward. The TUI installer asks a few use-case questions and applies appropriate settings; a unified dashboard gives a single view of system status.

| Layer | Components |
|---|---|
| Kernel hardening | KASLR, module signing, lockdown mode, hardened usercopy, sysctl hardening |
| Policy enforcement | SELinux, seccomp, cgroups, namespaces, capabilities |
| Network defense | nftables, CrowdSec, fail2ban |
| Network IPS | Suricata |
| Host runtime protection | Falco, Tetragon |
| Monitoring & SIEM | Wazuh, osquery |
| Malware scanning (evaluated) | ClamAV, LMD, planned custom scanner |

</details>

<details>
<summary><b>Configuration & ephemeral environments</b></summary>

System configuration lives in `/etc/nexis/` as schema-validated TOML:

```text
system.toml
packages.toml
services.toml
users.toml
profiles/
modules/
```

Ephemeral shell environments (similar to `nix-shell`) are planned.

</details>

---

## Community & Contact

Use [GitHub Issues](https://github.com/NexisOS) for discussion, feedback, and inquiries.

[Contributing](https://github.com/NexisOS/.github/blob/main/CONTRIBUTING.md) · [Code of Conduct](https://github.com/NexisOS/.github/blob/main/CODE_OF_CONDUCT.md) · [Security Policy](https://github.com/NexisOS/.github/blob/main/SECURITY.md) · [Governance](https://github.com/NexisOS/.github/blob/main/GOVERNANCE.md)

**Maintainer:** kyle.gortych.dev@gmail.com — sole maintainer; response times may vary.

**AI-assisted contributions** are welcome, held to the same standards as any other: human review is mandatory, contributors must understand what they submit, and generated code must be verified for correctness, security, and licensing.

---

## License & Legal

Licensed under **GNU GPL v3.0**, provided *as is* without warranty. NexisOS collects no user data, provides no hosted services, and is independently developed. It is not affiliated with the Linux Foundation, the NixOS Foundation, or any other organization; all referenced trademarks belong to their respective owners.

NexisOS does not implement jurisdiction-specific or speculative compliance mechanisms unless clearly required and technically well-defined. The maintainer has concerns about legislation (e.g., U.S. H.R. 8250) that would impose unclear or technically impractical requirements on decentralized, free and open-source operating systems, particularly where convenience-oriented mandates trade away user anonymity. Opinions expressed are the maintainer's own and do not affect the project's technical scope.
