# AGENTS.md - peakcode

This file provides guidance to AI coding agents working in this repository.

peakcode is an AI coding agent for the terminal, written in Rust. It is modeled on
[opencode](https://opencode.ai), not on Claude Code. The default terminal background
is `none` (the terminal decides the background, peakcode never forces one).

## Naming convention

The project name is **`peakcode`**, always lowercase, never `Peakcode`, `PeakCode`,
`PEAKCODE`, or any other casing. This applies to docs, code comments, commit messages,
PR descriptions, READMEs, CHANGELOGs, CLI output, binary names, and prose everywhere.

The binary is `peakcode`, the crate is `peakcode`, the config directory is `peakcode`.
Repo slugs already follow this; extend it to all prose.

## ALWAYS `git pull` before starting work

Before editing, building, or reasoning about this repo, run `git pull --ff-only`
(or `git fetch` + check). Local checkouts can drift behind `origin/main`; editing
stale code wastes work because changes conflict on rebase.

## Architecture decisions: build for the standard, not the corner shop

When making any architecture decision, design it as if this system is meant to
compete with the best-in-class tools and become a new standard, not to out-do a toy.
Reach for scalable, production-grade solutions over toy ones: pick the option that
still holds up at scale, under load, and in front of real users. Aim high by default.

## Documentation Conventions

When modifying a module, check if documentation for it exists under `/docs/`. If a
matching doc file exists, update it to reflect the changes. If no doc file exists, do
not create one.

Any change that touches a public contract - the provider/LLM interface, the tool-calling
API, the MCP integration, the plugin/tool interface, or anything users or extensions
build against - **must** be accompanied by a documentation update in `/docs/`. If no doc
file exists yet, create one. Treat these docs the same way you would treat a public
library API: every addition, removal, or behavioral change must be reflected in the docs
before the change is considered complete.

## Documentation authoring (`/docs`)

The peakssh docs platform imports each repository's `/docs` folder into the central docs
backend (docs.peakssh.dev). Author docs so the importer maps them correctly:

- **Scope:** only the repository's `/docs` folder is imported. Anything outside `/docs`,
  including the repo's top-level `README.md`, is ignored. A `README.md` or `index.md`
  that lives *inside* `/docs` is imported as an ordinary document; those names get no
  special treatment.
- **Folders = categories, `.md` files = documents.** A subfolder becomes a subcategory;
  each `.md` file becomes a document under its folder's category.
- **Only `.md` files and subfolders** are considered inside `/docs`. Any other file type
  is ignored by the importer.
- **Ordering:** prefix a file or folder with `NN-` (e.g. `01-intro.md`, `02-setup/`) to
  set its navigation position. The numeric prefix is stripped from the slug
  (`01-intro.md` -> slug `intro`). Without a prefix, items are ordered alphabetically.
- **Slugs** derive from the file/folder name (lowercased, non-alphanumerics -> hyphens)
  after removing any `NN-` prefix and the `.md` extension. Keep names slug-friendly:
  `^[a-z0-9]+(?:-[a-z0-9]+)*$`.
- **Document title** resolves in order: YAML frontmatter `title:` -> first `# H1`
  heading -> file name. A leading frontmatter block (`--- ... ---`) is used only for the
  title and is stripped from stored content.
- **Category title** is humanized from the folder name (`provider-integration` ->
  "Provider Integration").
- One document per file; max content size is 512 KB (larger files are skipped).
- A category that **tracks** a repo is managed by that repo: its documents are overwritten
  on each sync. Edit docs in the repo, not in the admin panel.

Configure import/tracking in the admin panel: **/docs/documents -> "Import documents"**.

## Git Conventions

- **Commits:** Always use `git commit -s` (adds `Signed-off-by` trailer, a Developer
  Certificate of Origin attestation).
- **Commit messages:** Follow [Conventional Commits](https://www.conventionalcommits.org/):
  `<type>(<scope>): <description>` (e.g. `feat(provider): add OpenAI streaming`).

## Style

- Never use the em dash (Unicode U+2014) in any text: code comments, docs, commit
  messages, and CLI output. Use a normal hyphen `-`, a comma, or rephrase.
- Default background is `none`: peakcode must never force a terminal background color.
  It adapts to whatever the terminal reports. Themes/colors are opt-in, never imposed.
