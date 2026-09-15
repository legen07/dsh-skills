<p align="center">
  <h3 align="center">dsh-skills</h3>
  <p align="center">A curated collection of DeepSeek Harness (DSH) skills for AI agents.</p>

  <p align="center">
    <a href="https://github.com/legen07/dsh-skills/actions">
      <img src="https://img.shields.io/badge/maintenance-active-brightgreen" alt="Maintenance Status">
    </a>
    <a href="https://github.com/legen07/dsh-skills/blob/main/LICENSE">
      <img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License">
    </a>
    <a href="https://github.com/legen07/dsh-skills/stargazers">
      <img src="https://img.shields.io/badge/Stars-0-E34C26?logo=github" alt="Stars">
    </a>
    <a href="https://github.com/legen07/dsh-skills/pulses">
      <img src="https://img.shields.io/github/commit-activity/m/legen07/dsh-skills" alt="Commit Activity">
    </a>
  </p>
</p>

---

## 🎯 About

**dsh-skills** is a repository of custom skills for [DeepSeek Harness (DSH)](https://dsh.ai) — an AI agent platform that allows developers to extend their agents with reusable, task-specific instructions called *skills*.

Each skill is a self-contained Markdown file (`SKILL.md`) that provides an AI agent with domain-specific knowledge, workflows, and best practices. Skills are loaded on-demand and applied contextually when the agent encounters relevant tasks.

## 📦 Skills

| Skill | Description |
|-------|-------------|
| **[apple-design](/apple-design/SKILL.md)** | Apple's approach to interface design and fluid, physical motion translated for the web. Covers gesture-driven UI, spring animations, drag/swipe interactions, translucent materials, and motion design principles. |
| **[nextjs-16-edge](/nextjs-16-edge/SKILL.md)** | Production-grade Next.js 16+ with Bun, Turbopack, Biome, and Cloudflare Edge. Enforces strict 9-layer architecture, WAAPI animations, modular CSS, and fully static site generation. |
| **[website-production-hardening](/website-production-hardening/SKILL.md)** | Audits and remediates websites to production-grade quality across performance, SEO, accessibility, security, UX, legal compliance, and analytics. Produces verified, launch-ready sites. |

## 🚀 Quick Start

1. **Clone the repo:**
   ```bash
   git clone git@github.com:legen07/dsh-skills.git
   cd dsh-skills
   ```

2. **Install skills into DSH:**
   ```bash
   # Copy skills to the DSH skills directory
   for skill in */; do
     cp "$skill/SKILL.md" "$HOME/.dsh/skills/$skill/"
   done
   ```

3. **Configure your DSH agent** to load skills from this repository. Each skill is automatically discovered by name.

## 🏗️ Architecture

Each skill follows a consistent structure:

```
skill-name/
├── SKILL.md          # Core skill definition (YAML frontmatter + instructions)
└── ...               # Additional assets, templates, or resources (if any)
```

The `SKILL.md` file contains:
- **YAML frontmatter** — Metadata (name, description, version, category, tags)
- **Instructions** — Detailed task-specific guidance for the AI agent
- **Anti-patterns** — Common pitfalls to avoid
- **Checklists** — Implementation verification steps

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](/CONTRIBUTING.md) for guidelines.

## 📄 License

This project is licensed under the [MIT License](/LICENSE).

---

**Built with ❤️ by [Joe Legen](https://github.com/legen07)**

<p align="center">
  <a href="https://github.com/legen07/dsh-skills">
    <img src="https://profile-counter.glitch.me/legen07/dsh-skills/core.svg" alt="Joe Legen">
  </a>
</p>
