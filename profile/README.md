<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

## 🐧 Overview

**NexisOS** is a free and open-source Linux distribution designed to offer complete transparency, control, and reproducibility. Built on **OS-Tree** and leveraging a custom **Rust-based system level declarative package manager**, NexisOS introduces a fully declarative configuration system inspired by **NixOS** and **Artix Linux**, but with a unique approach that blends both flexibility and simplicity.

At the core of NexisOS is a **strictly declarative package management** system that integrates seamlessly with the system configuration, utilizing **TOML** files for managing packages and system settings. This unified approach ensures that users can declare both their package dependencies and configuration changes declaratively, reducing the risk of configuration drift and ensuring reproducible system setups.

Unlike traditional Linux distributions, NexisOS does not rely on central repositories, which helps minimize long-term maintenance costs. Instead, the user defines their own flake sources and configurations, bringing a "captain of your own ship" philosophy to system management.

Security is a foundational priority for NexisOS. The distribution ships with several pre-configured security tools such as **firewalld**, **ClamAV**, **Maldet**, **Falco**, and **Suricata** for endpoint protection, malware detection, runtime security monitoring, and intrusion prevention, helping to create secure and resilient systems right out of the box.

---

## 🔽 Download ISO

You can access the NexisOS project page on SourceForge. Please note that while the link is live, the ISO has not yet been built. It will be added once the first build is ready:

👉 [Download NexisOS ISO](https://sourceforge.net/projects/nexisos/files/latest/download)

> ⚠️ *Note: The ISO versions in the range of v0.x.x are currently experimental and intended for testing and feedback. Expect rapid iteration and updates.*

---

## 🚧 Project Status

NexisOS is currently in the **design and planning phase**. Development will begin after core architectural decisions are finalized and trademark and legal preparations are complete.

### In progress:
- Defining the distribution’s scope, goals, and values
- Applying appropriate licensing to repositories
- Designing the TOML-based configuration system
- Developing security tooling and package infrastructure
- Planning for long-term sustainability and funding

Stay tuned. Star or watch the repo to follow progress as it evolves.

---

## 🔍 Design Inspiration

NexisOS draws from the best parts of:

- **[NixOS](https://nixos.org)** for declarative system configuration, atomic upgrades, and reproducible builds
- **[Artix Linux](https://artixlinux.org)** for its minimalist, systemd-free architecture and alternative init systems

Unlike Artix, NexisOS will use **[Dinit](https://github.com/dimitri/dinit)** as its default init system, chosen for its simplicity, modern design, and clean dependency model.

Our goal is to merge these concepts into a focused operating system built on clarity, reproducibility, and user control.

---

## ⚖️ Disclaimer

This is an independent community project. It is **not officially affiliated with**, **endorsed by**, or **sponsored by** the Linux Foundation, NixOS Foundation, or Artix Linux team.

---

## 📂 Repositories

Explore the NexisOS GitHub organization to find core components of the distribution, including:

- Declarative configuration modules
- ISO building scripts
- Package overlay definitions
- System documentation

---

## 🙏 Acknowledgments

NexisOS is built upon the contributions of the open-source community. Special thanks to:

    The NixOS community for pioneering declarative system configurations, atomic upgrades, and reproducible builds, which directly inspired NexisOS’s design.

    The Artix Linux team for their systemd-free, minimalist approach to Linux, which influenced our choice of Dinit as the default init system.

    OS-Tree for providing the powerful system image management framework at the core of NexisOS’s declarative package management.

    Dinit for its simplicity, modern design, and clean dependency model, which enables flexible and efficient service management.

    The Rust community for developing the language that powers NexisOS’s declarative package manager, chosen for its performance and safety.

    The security-focused open-source projects (e.g., firewalld, ClamAV, Maldet, Falco, Suricata) for tools that enhance NexisOS’s built-in security.

    The broader Linux and open-source communities for their invaluable contributions to the tools and libraries that make projects like NexisOS possible.

NexisOS would not be possible without the foundational work of the open-source ecosystem. The project is deeply grateful for the contributions that have made this work possible, and I, as the founder of NexisOS, personally appreciate the support and inspiration provided by all involved.

As development progresses, we look forward to collaborating with future maintainers and contributors to build on this foundation and create a robust and sustainable operating system for all.

---

## 📢 Community & Support

NexisOS is a personal project led and developed by a single maintainer. Community channels, including forums and chat, are coming soon.

In the meantime:

- Please use [GitHub Issues](https://github.com/NexisOS/issues) to report bugs, request features, or share ideas.
- Discussions will be enabled shortly to facilitate design conversations and collaboration.
- Stay tuned for announcements about the official community chat (likely on Matrix, Discord, or IRC).

Your feedback and participation are highly appreciated and essential to shaping NexisOS’s future. Thank you for your patience and support during this early phase!

---

## 📬 Contact the Creator

If you'd like to share ideas, provide feedback, or get in touch directly, feel free to email me at:

**[kyle.gortych.dev@gmail.com](mailto:kyle.gortych.dev@gmail.com)**

Please note that as the sole maintainer, response times may vary. For general issues and feature requests, we encourage using GitHub Issues or Discussions once available.

---
