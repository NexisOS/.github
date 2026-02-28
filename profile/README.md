<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

# 🐧 Overview

NexisOS is a free and open-source Linux distribution focused on transparency, control, reproducibility, and high performance through declarative configuration. Built with a custom Rust-based package manager using TOML configuration files, NexisOS offers a NixOS-inspired approach with key design improvements for greater flexibility, performance, and integration with Linux standards.

## Common NixOS pain points
While NixOS introduced groundbreaking ideas for reproducibility and declarative system management, it has several limitations:

Full derivation hashing: Nix couples builds to environment, compiler paths, and derivation metadata. Small unrelated changes can trigger large rebuilds. Binary reuse is difficult across machines or over time.

Slow store optimization: Deduplication and rebuild optimizations require global scans (nix-store --optimize) that are I/O and CPU heavy.

SELinux incompatibility: The immutable /nix/store path model makes SELinux labeling complex and inconsistent.

Nebulous conventions: Nix DSL, module composition, and derivation standards often conflict with traditional Linux filesystem hierarchy and conventions.

Limited flexibility: Mutable directories, virtual filesystems, and work directories require workarounds like nix-ld.

Debuging failed rebuilds: Doesn't directly show what package caused a transitive dependency to fail.

NexisOS was designed to overcome these issues with a system that:

Works with Linux, not against it.

Supports high-performance incremental rebuilds.

Supports SELinux natively.

Allows safe binary reuse.

Follows clear Linux conventions and deterministic identity models.

Minimizes unnecessary rebuilds and rebuild propagation.

### Key Design Principles

note to add RocksDB due to parallel writes for multi-host rebuilds

Declarative Configuration: System and package management through TOML files—no DSL required.

Graph-Aware Package Identity: Package IDs based on hash(file content) + hash(dependency graph) rather than full derivation, allowing binary reuse and faster rebuilds.

Minimal Rollbacks: Generation symlink switching rather than full snapshots, enabling instant rollback.

Immutable Store / Projected FHS: CAS store with immutable binaries, and projected root filesystem (/system) for execution with proper SELinux labels.

High Performance: Insert-time deduplication, BLAKE3 hashing, memory-mapped database indexes, parallel builds, minimal I/O overhead.

Security-First: Pre-configured security stack (ClamAV, Maldet, Tetragon, Suricata, SELinux).

### Security Stack

| Security Layer  | Implementation                                         |
| :-------------- | :----------------------------------------------------- |
| Firmware / Boot | Secure Boot, TPM, dm-verity                            |
| Kernel          | SELinux, seccomp, LKRG, IMA/EVM, lockdown              |
| Runtime         | init sandboxing, Tetragon, ClamAV, R-FX Networks's LMD |
| Isolation       | namespaces, cgroups v2                                 |
| Network         | nftables, Suricata                                     |
| Filesystem      | read-only root with controlled mutability              |

---

## 🚧 Project Status

NexisOS is in **active design phase**. Development will begin after core architecture is finalized.

**Current Focus:**
- Finalizing package manager design and TOML configuration system
- Designing custom init system
- Security tooling integration
- Repository structure and build infrastructure

> ⚠️ **Note**: Once released, v0.x.x versions will be experimental and intended for testing.

---

## 📂 Repository Structure

The NexisOS project is split into two repositories:

1. **nexisos-installer** - Minimal TUI based installer ISO with WiFi support for system installation
2. **nexisos-packages** - Core distribution packages bootstrapped/downloaded by the installer

---

## 🔧 Init System

NexisOS uses a custom init system designed for modern Linux kernel features and declarative configuration integration. Key features include:

- **pidfd support**: Race-condition-free process management using file descriptors instead of PIDs
- **Declarative service definitions**: Services defined in TOML alongside system configuration
- **Modern kernel integration**: Built to leverage cgroups v2, namespaces, and other contemporary Linux features
- **Flexible dependency management**: Advanced service ordering and dependency resolution

This init system is designed to integrate seamlessly with NexisOS's declarative philosophy while providing robust, secure service management and systemd compatibility.

---

## 🔒 Filesystem Access Policy

NexisOS enforces immutability in /store (immutable CAS) and controlled projections for active generations.


| Directory                                   | Mutable | Notes                                        |
| :------------------------------------------ | :------ | :------------------------------------------- |
| `/store`                                    | ❌      | Immutable content-addressed store            |
| `/generations/<id>/rootfs`                  | ❌      | Projected FHS for execution, SELinux labeled |
| `/system`                                   | ❌      | Symlink to active generation                 |
| `/var/lib`                                  | Partial | Application data in TOML                     |
| `/var/log`, `/run`, `/tmp`                  | ✅      | Runtime and temporary data                   |
| `/home/<user>/.config`, `.local/share`      | ❌      | Declared in TOML only                        |
| `/home/<user>/Documents`, `Downloads`, etc. | ✅      | User-controlled directories                  |

---

## 🛠️ Package Manager Architecture

<details>
<summary>View Package Installation Flow Diagram</summary>

<div align="center" style="background-color: white; padding: 20px;">
  <img src="https://github.com/NexisOS/.github/blob/main/diagram1.svg" width="100%" alt="Package Install Flow">
</div>

</details>

### Key Features

Graph-Aware Identity: PackageID = blake3(file content) + blake3(dependency graph).

Enables binary reuse across machines and rebuilds.

Avoids unnecessary rebuild cascades.

Faster than Nix full derivation hashing.

Content-Addressed Store (CAS):

/store/
    /objects/aa/bb/<ContentID>
    /packages/aa/<PackageID>/manifest.toml

Two-level bucket structure for scale

Immutable objects, deduplicated on insert

Generation Projection:

/generations/<id>/rootfs
/system -> /generations/<id>/rootfs

Symlink switch enables minimal rollbacks (O(1))

Projected FHS layout with proper SELinux labels

Insert-Time Deduplication:

No global store scans

Deduplication occurs at file insertion

O(new files only)

Metadata Database:

LMDB / RocksDB / SQLite for ContentID → object path, PackageID → content list

Supports GC and fast rebuild checks

Performance Optimizations:

BLAKE3 hashing (parallelized)

Memory-mapped caches for dependency DAG

Parallel builds with lock-free queues

Hybrid chunking & compression for changed files (Zstd/LZ4)

Optional bloom filters / roaring bitmaps for very large stores

Distribution & Updates:

HTTP/3 transfers

BitTorrent/IPFS-style replication for binary packages

---

## 📥 Download

ISO builds will be available on SourceForge once the first release is ready:

👉 [NexisOS on SourceForge](https://sourceforge.net/projects/nexisos/files/latest/download)

---

## ⚖️ Disclaimer

This is an independent, community-driven project maintained by an individual developer currently. It is **not officially affiliated with** the Linux Foundation, NixOS Foundation, Artix Linux, or any other tools mentioned.

All trademarks are property of their respective owners and used for identification purposes only.

---

## 🙏 Acknowledgments

NexisOS builds upon the work of many open-source projects:

- **NixOS** for declarative system configuration and reproducible builds
- **Artix Linux** and **Dinit** for a more secure and efficient drop-in replacement for Systemd
- **Rust community** for the language powering the package manager
- **Security projects** firewalld, SELinux, ClamAV, R-FX Networks's LMD, Tetragon, Suricata for built-in security tools
- The broader **Linux and open-source communities** for foundational tools and libraries

---

## 📢 Community & Support

As a personal project by a single maintainer, community channels are coming soon.

**For now:**
- Use [GitHub Issues](https://github.com/NexisOS/issues) for bugs, features, and ideas
- GitHub Discussions will be enabled soon for design conversations
- Official community forum/chat still being decided

---

## 📬 Contact

**Email**: [kyle.gortych.dev@gmail.com](mailto:kyle.gortych.dev@gmail.com)

Please use GitHub Issues for general inquiries. As the sole maintainer, response times may vary.
