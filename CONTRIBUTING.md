# DSH Skills — CONTRIBUTING.md

## Contributing to dsh-skills

Thank you for your interest in contributing to **dsh-skills**! This document provides guidelines and instructions for contributing new skills or improvements to existing ones.

## 📋 Getting Started

1. **Fork the repository** and clone your fork locally.
2. **Create a feature branch** for your work:
   ```bash
   git checkout -b feat/skill-name
   ```

## 🛠️ Adding a New Skill

Each skill must follow the standard structure:

```
skill-name/
└── SKILL.md
```

### SKILL.md Requirements

The `SKILL.md` file must contain:

1. **YAML Frontmatter** with at minimum:
   ```yaml
   ---
   name: skill-name
   description: A concise description of what this skill enables.
   version: 1.0.0
   ---
   ```

2. **Clear Instructions** — Detailed, actionable guidance for the AI agent.
3. **Anti-patterns** — Common mistakes to avoid.
4. **Checklists** — Verification steps for implementation.

### Naming Conventions

- Use **kebab-case** for skill names (e.g., `my-awesome-skill`)
- Use **lowercase** directory names
- Descriptions should be concise and actionable

## 📝 Commit Guidelines

- Use descriptive commit messages: `feat: add [skill-name] skill`
- Follow [Conventional Commits](https://www.conventionalcommits.org/) format
- Keep commits focused on a single change

## ✅ Quality Checklist

Before submitting, ensure:

- [ ] `SKILL.md` has valid YAML frontmatter
- [ ] Description clearly explains the skill's purpose
- [ ] Instructions are detailed and actionable
- [ ] Anti-patterns are documented
- [ ] No hardcoded personal information
- [ ] `.gitignore` is respected (no `.dsh/`, `.env`, etc.)

## 🤝 Review Process

1. Submit a **Pull Request** with a clear description.
2. Maintainers will review for quality and consistency.
3. Address any feedback promptly.
4. Once approved, your skill will be merged.

## 📄 License

By contributing, you agree that your contributions will be licensed under the [MIT License](/LICENSE).

---

### Questions?

Open an [Issue](https://github.com/legen07/dsh-skills/issues) for any questions or discussions.
