# Contributing to S.T.A.T.

Thank you for your interest in contributing to S.T.A.T.!

This guide explains our contribution workflow, conventions, and review process.

---

## Before You Start

### Understanding Our Workflow

We use a **fork-first workflow**. This means:
- You work on your own copy (fork) of the repository
- Changes are proposed via Pull Requests from your fork
- This keeps the main repository clean and secure

### Why Fork-First?

1. **Security**: Only maintainers have write access to the main repo
2. **Experimentation**: You can freely experiment in your fork
3. **Learning**: Great practice for contributing to any open source project
4. **Backup**: Your fork serves as a backup of your work

---

## Fork-First Workflow

### Step 1: Fork the Repository

1. Click the "Fork" button on the repository page
2. This creates your personal copy at `github.com/YOUR-USERNAME/stat`

### Step 2: Clone Your Fork

```bash
# Clone your fork locally
git clone https://github.com/YOUR-USERNAME/stat.git
cd stat

# Add the original repo as "upstream" for syncing
git remote add upstream https://github.com/The-OASIS-Project/stat.git
```

### Step 3: Keep Your Fork Updated

```bash
# Fetch upstream changes
git fetch upstream

# Merge upstream main into your local main
git checkout main
git merge upstream/main

# Push updates to your fork
git push origin main
```

### Step 4: Create a Feature Branch

**Never work directly on `main`**. Always create a branch.

**First, verify the correct issue number:**

```bash
# List open issues to find the right issue number
gh issue list
```

**Then create your branch with that issue number:**

```bash
git checkout -b feat/stat/<issue#>-description

# Example: Working on issue #3
git checkout -b feat/stat/3-foundation-files
```

---

## Branch Naming Conventions

Use descriptive branch names following this pattern:

```
type/stat/<issue#>-short-description
```

### Branch Types

| Type | Purpose | Example |
|------|---------|---------|
| `feat/` | New feature | `feat/stat/5-new-sensor-driver` |
| `fix/` | Bug fix | `fix/stat/6-i2c-timeout` |
| `docs/` | Documentation only | `docs/stat/7-mqtt-topics` |
| `refactor/` | Code restructuring | `refactor/stat/8-battery-model` |
| `test/` | Adding/updating tests | `test/stat/9-bms-parsing` |
| `chore/` | Maintenance tasks | `chore/stat/10-update-deps` |

---

## Conventional Commits

We follow [Conventional Commits](https://www.conventionalcommits.org/) for clear, consistent history.

### Format

```
type(scope): description

[optional body]

[optional footer]
```

### Commit Types

| Type | When to Use |
|------|-------------|
| `feat` | Adding new functionality |
| `fix` | Fixing a bug |
| `docs` | Documentation changes only |
| `style` | Formatting (no code logic change) |
| `refactor` | Restructuring without behavior change |
| `test` | Adding or updating tests |
| `chore` | Maintenance (dependencies, configs) |

### Examples

```bash
# Feature
git commit -m "feat(bms): add Daly BMS cell balancing status"

# Bug fix
git commit -m "fix(ina238): handle I2C bus timeout gracefully"

# Documentation
git commit -m "docs(mqtt): document JSON payload format"
```

### Why Conventional Commits?

1. **Automatic changelogs**: Tools can generate release notes
2. **Clear history**: Easy to understand what changed and why
3. **Semantic versioning**: Commit types inform version bumps
4. **Better reviews**: Reviewers understand intent quickly

---

## Hardware Testing Notes

S.T.A.T. interacts directly with hardware sensors. When contributing:

### If You Have Hardware

- Test with actual I2C sensors (INA238, INA3221) when possible
- Verify MQTT publishing with `mosquitto_sub -t stat/#`
- Test BMS integration if you have a Daly Smart BMS
- Check systemd service operation with `journalctl -u oasis-stat`

### If You Don't Have Hardware

- Code changes that don't touch driver logic can be reviewed without hardware
- Use `i2cdetect` output examples in issues to describe expected behavior
- Document any assumptions about hardware behavior
- Mock hardware support is planned (see S.C.O.P.E. [meta-issue](https://github.com/malcolmhoward/the-oasis-project-meta-repo/issues/30) tracking)

---

## Suggesting Features

### Before Proposing

1. **Search existing issues** — your idea may already be discussed
2. **Check the roadmap** — it might be planned already
3. **Consider scope** — does it fit the project's goals?

### Priority Assessment

When proposing features, consider these factors:

| Factor | Questions to Ask |
|--------|------------------|
| **Impact** | How many users benefit? How significant is the improvement? |
| **Effort** | How complex is implementation? What's the maintenance burden? |
| **Risk** | What could break? Are there security implications? |
| **Alignment** | Does it fit project goals and architecture? |

---

## Code Review Process

### What Reviewers Look For

1. **Correctness**: Does the code do what it claims?
2. **Tests**: Are changes covered by tests?
3. **Style**: Does it follow project conventions?
4. **Documentation**: Are changes documented?
5. **Security**: Are there any vulnerabilities?
6. **Performance**: Any performance implications?

### Educational Review Criteria

We review with education in mind:

| Criteria | What We Check |
|----------|---------------|
| **Clarity** | Is the code readable and self-documenting? |
| **Simplicity** | Is this the simplest solution that works? |
| **Maintainability** | Will future contributors understand this? |
| **Best Practices** | Does it follow established patterns? |

---

## Pull Request Process

### Before Submitting

- [ ] Code compiles without warnings (`-Wall -Wextra -Wpedantic`)
- [ ] Tested on target hardware (or documented limitations)
- [ ] Branch is up-to-date with main
- [ ] Commit messages follow conventions
- [ ] Documentation updated if needed

### PR Description Template

```markdown
## Summary
Brief description of changes.

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation
- [ ] Refactoring

## Testing
How were changes tested? (hardware, platform, etc.)

## Checklist
- [ ] Compiles without warnings
- [ ] Tested on hardware (or N/A with explanation)
- [ ] Documentation updated
- [ ] No breaking changes (or documented)
```

---

## Coding Standards

### General Guidelines

- C99 standard
- 4-space indentation
- `snake_case` for functions and variables
- `UPPER_CASE` for constants and macros
- Write clear, self-documenting code
- Include comments for complex logic (especially I2C register operations)
- Follow existing patterns in the codebase
- Keep functions focused and small

---

## Code of Conduct

We are committed to providing a welcoming and inclusive environment.
Please be respectful and constructive in all interactions.

---

## Getting Help

- **Questions**: Open a discussion or issue
- **Bugs**: Open an issue describing the hardware, platform, and expected vs actual behavior
- **Features**: Open an issue with a feature request
