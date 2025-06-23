<div align="center">
  <img src="https://github.com/NexisOS/.github/blob/main/NexisOS2.png" width="50%" alt="NexisOS Logo">
</div>

## 🐧 Overview

**NexisOS** is a free and open-source Linux distribution that blends the philosophies and strengths of **NixOS** and **Artix Linux**. It is designed for users who value transparency, control, and reproducibility, offering a fully declarative configuration system and using **Dinit** as its init system.

At the heart of NexisOS is a strictly declarative package manager, inspired by **Nix flakes**, but with a more opinionated and streamlined design. Instead of supporting imperative package installations or multiple configuration styles, NexisOS embraces a single, consistent approach: packages are managed declaratively via **TOML** files. This design prioritizes simplicity, consistency, and reproducibility.

Unlike traditional distributions that maintain central package repositories, which can become maintenance heavy over time, NexisOS encourages a “**captain of your own ship**” philosophy. Users define and manage their own flake sources and configurations, reducing central complexity and promoting sustainability.

Security is a key priority. NexisOS ships with **ClamAV** (antivirus) and **Suricata** (IDS/IPS), preconfigured with sensible defaults to help users deploy secure systems out of the box.

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
<!--
## 🤝 How to Contribute

We welcome early input and discussion. Contributions are encouraged in the form of design suggestions, documentation, or architectural discussion.

Open an issue to propose ideas or improvements.

## 📬 Contact

We're currently setting up contact and community channels.

Until then:
- Use GitHub issues for feedback
- GitHub Discussions will be enabled soon
-->
