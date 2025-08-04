<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

## 🐧 Overview

**NexisOS** is a free and open-source Linux distribution that blends the philosophies and strengths of **NixOS** and **Artix Linux**. It is designed for users who value transparency, control, and reproducibility, offering a fully declarative configuration system and using **Dinit** as its init system.

At the heart of NexisOS is a strictly declarative package manager, inspired by **Nix flakes**, but with a more opinionated and streamlined design. Instead of supporting imperative package installations or multiple configuration styles, NexisOS embraces a single, consistent approach: packages are managed declaratively via **TOML** files. This design prioritizes simplicity, consistency, and reproducibility.

Unlike traditional distributions that maintain central package repositories, which can become maintenance heavy over time, NexisOS encourages a “**captain of your own ship**” philosophy. Users define and manage their own flake sources and configurations, reducing central complexity and promoting sustainability.

Security is a foundational priority for NexisOS. The distribution aims to provide comprehensive Linux security coverage by integrating a suite of endpoint protection tools and monitoring solutions. Currently, NexisOS plans to include and preconfigure security components such as **firewalld** (firewall management), **ClamAV** (antivirus), **Maldet** (Linux malware detector), **Falco** (runtime security and behavioral monitoring), and **Suricata** (intrusion detection and prevention system). These tools work together to provide layered, out-of-the-box protection, helping users deploy secure and resilient systems with ease.

---

## 🔽 Download ISO

You can try the latest ISO build of NexisOS by downloading it from SourceForge:

👉 [Download NexisOS ISO](https://sourceforge.net/projects/nexisos/files/latest/download)

> ⚠️ *Note: The ISO is currently experimental and intended for testing and feedback. Expect rapid iteration and updates.*

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

Special thanks to the **NixOS** and **Artix Linux** communities for their foundational work and open-source contributions. NexisOS would not be possible without them.

---

## 📢 Community & Support

NexisOS is a personal project led and developed by a single maintainer. Community channels, including forums and chat, are coming soon.

In the meantime:

- Please use [GitHub Issues](https://github.com/NexisOS/issues) to report bugs, request features, or share ideas.
- Discussions will be enabled shortly to facilitate design conversations and collaboration.
- Stay tuned for announcements about the official community chat (likely on Matrix, Discord, or IRC).

Your feedback and participation are highly appreciated and essential to shaping NexisOS’s future. Thank you for your patience and support during this early phase!

---

## 📬 Contact the Maintainer

If you would like to share ideas, provide feedback, or get in touch directly, you can email the maintainer of NexisOS at:

**[kyle.gortych.dev@gmail.com](mailto:kyle.gortych.dev@gmail.com)**

Please note that as the sole maintainer, response times may vary. For general issues and feature requests, we encourage using GitHub Issues or Discussions once available.

---
