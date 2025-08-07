<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

## 🐧 Overview

**NexisOS** is a free and open-source Linux distribution designed to offer complete transparency, control, and reproducibility. Leveraging a custom **Rust-based system level declarative package manager**, NexisOS introduces a fully declarative configuration system inspired by **NixOS** and **Artix Linux**, but with a unique approach that blends both flexibility and simplicity.

At the core of NexisOS is a **strictly declarative package management** system that integrates seamlessly with the system configuration, utilizing **TOML** files for managing packages and system settings. This unified approach ensures that users can declare both their package dependencies and configuration changes declaratively, reducing the risk of configuration drift and ensuring reproducible system setups.

Unlike traditional Linux distributions, NexisOS does not rely on central repositories, which helps minimize long-term maintenance costs. Instead, the user defines their own flake sources and configurations, bringing a "captain of your own ship" philosophy to system management.

Security is a foundational priority for NexisOS. The distribution ships with several pre-configured security tools such as **firewalld**, **ClamAV**, **Maldet**, **Falco**, and **Suricata** for endpoint protection, malware detection, runtime security monitoring, and intrusion prevention, helping to create secure and resilient systems right out of the box.

---

<details>
<summary>🔽 Download ISO</summary>

You can access the NexisOS project page on SourceForge. Please note that while the link is live, the ISO has not yet been built. It will be added once the first build is ready:

👉 [Download NexisOS ISO](https://sourceforge.net/projects/nexisos/files/latest/download)

> ⚠️ *Note: The ISO versions in the range of v0.x.x are currently experimental and intended for testing and feedback. Expect rapid iteration and updates.*

</details>

<details>

<summary>🚧 Project Status</summary>

NexisOS is currently in the **design and planning phase**. Development will begin after core architectural decisions are finalized and trademark and legal preparations are complete.

### In progress:
- Defining the distribution’s scope, goals, and values
- Applying appropriate licensing to repositories
- Designing the TOML-based configuration system
- Developing security tooling and package infrastructure
- Planning for long-term sustainability and funding

Stay tuned. Star or watch the repo to follow progress as it evolves.

### 🔒 Filesystem Mutability and Access Policy Overview

| Directory                         | Mutable?       | Notes                                         |
|----------------------------------|----------------|-----------------------------------------------|
| `/`                              | ❌             | Immutable root filesystem                      |
| `/etc`                           | ❌             | All config declared in TOML                    |
| `/usr`, `/lib*`, `/bin`, `/sbin` | ❌             | System binaries, read-only                      |
| `/var/lib`                       | ❌ or partially | App data should be declared if needed         |
| `/var/log`, `/run`, `/tmp`       | ✅             | Runtime and temp data — safe to be writable   |
| `/home/<user>`                   | ✅ partial     | Personal files allowed; config disallowed      |
| `/home/<user>/.config`           | ❌             | Declared in TOML only                          |
| `/home/<user>/.local/share`      | ❌             | Declared in TOML only                          |
| `/home/<user>/Downloads, Documents, etc.` | ✅     | User-controlled                               |

</details>

<details>

<summary>🔍 Design Inspiration</summary>

NexisOS draws from the best parts of:

- **[NixOS](https://nixos.org)** for declarative system configuration, atomic upgrades, and reproducible builds
- **[Artix Linux](https://artixlinux.org)** for its minimalist, systemd-free architecture and alternative init systems

Unlike Artix, NexisOS will use **[Dinit](https://github.com/dimitri/dinit)** as its default init system, chosen for its simplicity, modern design, and clean dependency model.

Our goal is to merge these concepts into a focused operating system built on clarity, reproducibility, and user control.

</details>

<details>

<summary>⚖️ Disclaimer</summary>

This is an independent, community-driven project, currently maintained by an individual developer. It is **not officially affiliated with, endorsed by, or sponsored by**:

- The Linux Foundation,
- NixOS Foundation,
- Artix Linux project,
- Dinit maintainers,
- or the authors of any additional tools or packages used in NexisOS (e.g., firewalld, ClamAV, Maldet, Falco, Suricata).

All product names, trademarks, and registered trademarks are property of their respective owners. Their use in this project is for identification, compatibility, and reference only, and does not imply any official endorsement.

NexisOS builds upon and is inspired by the work of many open source projects and contributors. Please see the [Acknowledgments](#acknowledgments) section for details.

</details>

<details>

<summary>📂 Repositories</summary>

Explore the NexisOS GitHub organization to find core components of the distribution, including:

- Declarative configuration modules
- ISO building scripts
- Package overlay definitions
- System documentation

</details>

<details>

<summary>🙏 Acknowledgments</summary>

NexisOS is built upon the contributions of the open-source community. Special thanks to:

- The NixOS community for pioneering declarative system configurations, atomic upgrades, and reproducible builds, which directly inspired NexisOS’s design.
- The Artix Linux team for their systemd-free, minimalist approach to Linux, which influenced our choice of Dinit as the default init system.
- Dinit for its simplicity, modern design, and clean dependency model, which enables flexible and efficient service management.
- The Rust community for developing the language that powers NexisOS’s declarative package manager, chosen for its performance and safety.
- The security-focused open-source projects (e.g., firewalld, ClamAV, Maldet, Falco, Suricata) for tools that enhance NexisOS’s built-in security.
- The broader Linux and open-source communities for their invaluable contributions to the tools and libraries that make projects like NexisOS possible.

NexisOS would not be possible without the foundational work of the open-source ecosystem. The project is deeply grateful for the contributions that have made this work possible, and I, as the founder of NexisOS, personally appreciate the support and inspiration provided by all involved.

As development progresses, we look forward to collaborating with future maintainers and contributors to build on this foundation and create a robust and sustainable operating system for all.

</details>

<details>

<summary>📢 Community & Support</summary>

NexisOS is a personal project led and developed by a single maintainer. Community channels, including forums and chat, are coming soon.

In the meantime:

- Please use [GitHub Issues](https://github.com/NexisOS/issues) to report bugs, request features, or share ideas.
- Discussions will be enabled shortly to facilitate design conversations and collaboration.
- Stay tuned for announcements about the official community chat (likely on Matrix, Discord, or IRC).

Your feedback and participation are highly appreciated and essential to shaping NexisOS’s future. Thank you for your patience and support during this early phase!

</details>

<details>

<summary>📬 Contact the Creator</summary>

If you'd like to share ideas, provide feedback, or get in touch directly, feel free to email me at:

**[kyle.gortych.dev@gmail.com](mailto:kyle.gortych.dev@gmail.com)**

Please note that as the sole maintainer, response times may vary. For general issues and feature requests, we encourage using GitHub Issues or Discussions once available.

</details>
