<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

# 🐧 Overview

NexisOS is a free and open-source Linux distribution focused on **transparency, control, reproducibility, and high performance** through declarative configuration. Built with a custom Rust-based package manager using TOML configuration files, NexisOS offers a NixOS-inspired approach with key design improvements for greater flexibility, performance, and integration with Linux standards.

> ⚠️ **Project Status**: NexisOS is in **active design phase**. Development begins after core architecture is finalized. Once released, v0.x.x versions will be experimental and intended for testing.

---

## 💡 Motivation: Why Not NixOS?

<details>
<summary>Common NixOS pain points NexisOS addresses</summary>

| Pain Point | Description |
| :--------- | :---------- |
| Full derivation hashing | Nix couples builds to environment, compiler paths, and derivation metadata. Small unrelated changes trigger large rebuilds; binary reuse is difficult across machines or over time. |
| Slow store optimization | Deduplication requires global scans (`nix-store --optimize`) that are I/O and CPU heavy. |
| SELinux incompatibility | The immutable `/nix/store` path model makes SELinux labeling complex and inconsistent. |
| Nebulous conventions | Nix DSL, module composition, and derivation standards often conflict with traditional Linux filesystem hierarchy. |
| Limited flexibility | Mutable directories, virtual filesystems, and work directories require workarounds like `nix-ld`. |
| Opaque rebuild failures | Doesn't directly show what package caused a transitive dependency to fail. |

</details>

### Key Design Principles

<!-- note: add RocksDB due to parallel writes for multi-host rebuilds -->

- **Declarative Configuration** — System and package management through TOML files; no DSL required.
- **Graph-Aware Package Identity** — `PackageID = blake3(file content) + blake3(dependency graph)`, enabling binary reuse and faster rebuilds.
- **Minimal Rollbacks** — Generation symlink switching (O(1)) rather than full snapshots.
- **Immutable Store / Projected FHS** — CAS store with immutable binaries and a projected root filesystem (`/system`) with proper SELinux labels.
- **High Performance** — Insert-time deduplication, BLAKE3 hashing, memory-mapped indexes, parallel builds, minimal I/O overhead.
- **Security-First** — Pre-configured security stack (ClamAV, Maldet, Tetragon, Suricata, SELinux).

---

## 🔒 Security Stack

<details>
<summary>View full security stack</summary>

| Layer | Implementation |
| :---- | :------------- |
| Firmware / Boot | Secure Boot, TPM, dm-verity |
| Kernel | SELinux, seccomp, LKRG, IMA/EVM, lockdown |
| Runtime | Init sandboxing, Tetragon, ClamAV, R-FX Networks's LMD |
| Isolation | Namespaces, cgroups v2 |
| Network | nftables, Suricata |
| Filesystem | Read-only root with controlled mutability |

</details>

---

## 🔧 Init System

NexisOS uses a custom init system built for modern Linux kernel features and declarative configuration. Highlights include **pidfd-based process management** (race-condition-free), **TOML-defined service declarations**, cgroups v2 / namespace integration, advanced dependency resolution, and systemd compatibility.

---

## 🛠️ Package Manager Architecture

<details>
<summary>View package installation flow diagram</summary>

<div align="center" style="background-color: white; padding: 20px;">
  <img src="https://github.com/NexisOS/.github/blob/main/diagram1.svg" width="100%" alt="Package Install Flow">
</div>

</details>

<details>
<summary>Graph-Aware Identity</summary>

`PackageID = blake3(file content) + blake3(dependency graph)`

- Enables binary reuse across machines and rebuilds
- Avoids unnecessary rebuild cascades
- Faster than Nix full derivation hashing

</details>

<details>
<summary>Content-Addressed Store (CAS)</summary>

```
/store/
    /objects/aa/bb/<ContentID>
    /packages/aa/<PackageID>/manifest.toml
```

- Two-level bucket structure for scale
- Immutable objects, deduplicated on insert

</details>

<details>
<summary>Generation Projection & Rollback</summary>

```
/generations/<id>/rootfs
/system -> /generations/<id>/rootfs
```

- Symlink switch enables O(1) rollbacks
- Projected FHS layout with proper SELinux labels

</details>

<details>
<summary>Performance Optimizations</summary>

- **Insert-time deduplication** — No global store scans; O(new files only)
- **BLAKE3 hashing** — Parallelized content hashing
- **Memory-mapped caches** for dependency DAG traversal
- **Parallel builds** with lock-free queues
- **Hybrid chunking & compression** (Zstd/LZ4) for changed files
- **Optional bloom filters / roaring bitmaps** for very large stores
- **Metadata DB** — LMDB / RocksDB / SQLite for ContentID → object path, PackageID → content list; supports GC and fast rebuild checks

</details>

<details>
<summary>Distribution & Updates</summary>

- HTTP/3 transfers
- BitTorrent/IPFS-style replication for binary packages

</details>

---

## 🔒 Filesystem Access Policy

<details>
<summary>View directory mutability table</summary>

| Directory | Mutable | Notes |
| :-------- | :------ | :---- |
| `/store` | ❌ | Immutable content-addressed store |
| `/generations/<id>/rootfs` | ❌ | Projected FHS for execution, SELinux labeled |
| `/system` | ❌ | Symlink to active generation |
| `/var/lib` | Partial | Application data in TOML |
| `/var/log`, `/run`, `/tmp` | ✅ | Runtime and temporary data |
| `/home/<user>/.config`, `.local/share` | ❌ | Declared in TOML only |
| `/home/<user>/Documents`, `Downloads`, etc. | ✅ | User-controlled directories |

</details>

---

## 📂 Repository Structure

| Repository | Purpose |
| :--------- | :------ |
| **nexisos-installer** | Minimal TUI-based installer ISO with WiFi support |
| **nexisos-packages** | Core distribution packages bootstrapped/downloaded by the installer |

---

## 📥 Download

ISO builds will be available on SourceForge once the first release is ready:

👉 [NexisOS on SourceForge](https://sourceforge.net/projects/nexisos/files/latest/download)

---

## 🙏 Acknowledgments

<details>
<summary>Projects that inspired or support NexisOS</summary>

- **NixOS** — Declarative system configuration and reproducible builds
- **Artix Linux** & **Dinit** — Secure and efficient drop-in Systemd replacement
- **Rust community** — Language powering the package manager
- **Security projects** — firewalld, SELinux, ClamAV, R-FX Networks's LMD, Tetragon, Suricata
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
This is an independent, community-driven project maintained currently by an individual developer. It is not affiliated with the Linux Foundation, NixOS Foundation, Artix Linux, or any other referenced projects. All trademarks belong to their respective owners and are used for identification only.

This Distribution is released under the GNU General Public License version 3.0 and is under active development. Features may be incomplete or experimental.

As provided under the GNU General Public License version 3.0, the software is supplied “as is,” without any warranty, express or implied, including merchantability or fitness for a particular purpose. All risk regarding quality, performance, compliance, and deployment remains with the user.

This project publishes software code only. The maintainer does not operate an online service or platform and does not collect, process, or monitor end-user data through the Distribution.

Certain jurisdictions — including the State of California and other U.S. states or foreign countries — may impose age verification, identity controls, monitoring, or similar regulatory requirements at the operating system level. This Distribution does not currently implement such mechanisms, and no widely adopted cross-architecture standard exists for doing so across the diverse platforms it supports (including x86_64, ARM, virtual, cloud, embedded, and custom systems). If a clear and technically viable standard emerges, it may be evaluated for incorporation.

Users, integrators, and redistributors are solely responsible for ensuring compliance with all applicable laws. If you reside in or operate within a jurisdiction requiring such controls, seek qualified legal counsel before installing or distributing this software. If compliance cannot be ensured, do not install, distribute, or deploy the Distribution.
