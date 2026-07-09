# Contributing to peakcode

Thank you for your interest in contributing to peakcode! peakcode is an AI coding agent
for the terminal, written in Rust, and a product of the [peakssh](https://peakssh.dev)
ecosystem. Every contribution - big or small - is welcome and appreciated.

---

## Table of Contents

- [Getting Started](#getting-started)
- [How to Contribute](#how-to-contribute)
- [Commit Message Convention](#commit-message-convention)
- [Pull Request Guidelines](#pull-request-guidelines)
- [Tech Stack Overview](#tech-stack-overview)
- [Reporting Issues](#reporting-issues)

---

## Getting Started

1. **Fork** this repository and clone it locally.
2. Create a new branch from `main`:
   ```bash
   git checkout -b feat/your-feature-name
   ```
3. Make your changes, then push your branch and open a Pull Request.

> Read this repo's `README.md` for local setup and build instructions before diving in.

---

## How to Contribute

- **Bug fixes** - Found something broken? Open an issue first, then submit a fix.
- **New features** - Discuss larger ideas via an issue before writing code.
- **Docs & typos** - Always welcome, no issue needed.
- **Refactoring** - Keep it focused and well-explained in the PR description.

---

## Commit Message Convention

We follow the [Conventional Commits](https://www.conventionalcommits.org/) standard.
This keeps the history clean and makes changelogs easy to generate.

### Sign off your commits

All commits **must** be signed off with `git commit -s` (adds a `Signed-off-by:` trailer).
This is a Developer Certificate of Origin attestation that you have the right to submit
the change under the project's license.

### Format

```
<type>(<scope>): <short description>
```

### Types

| Type       | When to use                                      |
|------------|--------------------------------------------------|
| `feat`     | A new feature                                    |
| `fix`      | A bug fix                                        |
| `docs`     | Documentation changes only                       |
| `style`    | Formatting, missing semicolons, etc. (no logic)  |
| `refactor` | Code change that is neither a fix nor a feature  |
| `test`     | Adding or updating tests                         |
| `chore`    | Build process, dependencies, tooling             |
| `ci`       | CI/CD configuration changes                      |

### Examples

```
feat(provider): add OpenAI streaming support
fix(tui): correct input rendering on resize
docs(readme): update local setup instructions
chore(deps): bump tokio to 1.40
```

> **Scope** is optional but encouraged - use the module or component name.

---

## Pull Request Guidelines

- **One PR = one concern.** Don't mix unrelated changes.
- Fill in the PR description: what changed and why.
- Link related issues with `Closes #123` or `Relates to #456`.
- Keep PRs reasonably small - easier to review, faster to merge.
- Make sure existing functionality isn't broken before submitting (`cargo build`,
  `cargo test`, `cargo clippy`).
- A maintainer will review your PR as soon as possible. Be patient and responsive to
  feedback.

### PR Title

Your PR title should follow the same convention as commit messages:

```
feat(provider): add streaming token support
```

---

## Tech Stack Overview

| Layer          | Technology      |
|----------------|-----------------|
| Language       | Rust            |
| Async runtime  | tokio           |
| LLM providers  | OpenAI (first)  |

Make sure you have a recent Rust toolchain installed (`rustup`).

---

## Reporting Issues

When opening a bug report, please include:

- A clear and descriptive title.
- Steps to reproduce the problem.
- Expected vs. actual behavior.
- Environment details (OS, terminal, Rust version, provider/model used).

For feature requests, describe the use case and why it would benefit the project.

---

Thank you for helping make peakcode better!
