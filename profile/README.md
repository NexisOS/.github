<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

## 🐧 Overview

**NexisOS** is a free and open-source Linux distribution focused on transparency, control, and reproducibility through declarative configuration. Built with a custom **Rust-based package manager** using **TOML** configuration files, NexisOS offers a NixOS-inspired approach with key design differences for greater flexibility and sustainability.

### Key Design Principles

- **Declarative Configuration**: System and package management through TOML files—no DSL required like nix syntax and reinventing the wheel for package configs
- **Decentralized Package Model**: No central repositories beyond core packages—users define their own sources
- **Flexible Mutability**: Support for mutable work directories and virtual filesystems e.g., Java security directories without workarounds like nix-ld
- **Security-First**: Pre-configured with a sensible security stack—ClamAV, Maldet, Tetragon, and Suricata

| Security Layer  | Implementation                                         |
|:----------------|:-------------------------------------------------------|
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

1. **nexisos-installer** - Whiptail-based installer ISO with WiFi support for system installation
2. **nexisos-packages** - Core distribution packages bootstrapped/downloaded by the installer

---

## 🔧 Init System

NexisOS uses a custom init system designed for modern Linux kernel features and declarative configuration integration. Key features include:

- **pidfd support**: Race-condition-free process management using file descriptors instead of PIDs
- **Declarative service definitions**: Services defined in TOML alongside system configuration
- **Modern kernel integration**: Built to leverage cgroups v2, namespaces, and other contemporary Linux features
- **Flexible dependency management**: Advanced service ordering and dependency resolution

This init system is designed to integrate seamlessly with NexisOS's declarative philosophy while providing robust, secure service management.

---

## 🔒 Filesystem Access Policy

NexisOS enforces immutability using SELinux or `chattr +i`. A base manifest tracks filesystem state, and generations are compared during rebuilds.

| Directory                                     | Mutable | Notes                         |
|:----------------------------------------------|:--------|:------------------------------|
| `/`, `/etc`, `/usr`, `/lib*`, `/bin`, `/sbin` | ❌      | Immutable system files        |
| `/var/lib`                                    | Partial | Application data in TOML      |
| `/var/log`, `/run`, `/tmp`                    | ✅      | Runtime and temporary data    |
| `/home/<user>/.config`, `.local/share`        | ❌      | Declared in TOML only         |
| `/home/<user>/Documents`, `Downloads`, etc.   | ✅      | User-controlled directories   |

---

## 🛠️ Package Manager Architecture

<details>
<summary>View Package Installation Flow Diagram</summary>

<div align="center" style="background-color: white; padding: 20px;">
  <img src="https://github.com/NexisOS/.github/blob/main/diagram1.svg" width="100%" alt="Package Install Flow">
</div>

</details>

The package manager is being designed with the following considerations:

- **Configuration**: TOML files (no DSL); native config file formats supported
- **Performance Optimizations**: Blake3 hashing, improved deduplication on write, cached IR with mmap, persistent DAG for dependencies
- **Data Structures**: SwissTable-style hash tables, succinct tries, bloom filters, ROAR bitmaps for feature flags
- **Chunking**: Hybrid fixed + rolling for changed files, Zstandard compression, LZ4 for metadata
- **Builds**: Hermetic sandboxed parallel builds, Chase-Lev work stealing queues, lock-free dependency traversal
- **Distribution**: HTTP/3 transfers, BitTorrent/IPFS-style replication

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
