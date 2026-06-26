# Superset-Dev Project Guide

This is a unified development workspace for building and distributing the Superset platform as a standalone product. It combines the vendored upstream monorepo, the native Rust backend, and the extracted TypeScript CLI into a single cohesive project.

## Project Layout
- `vendor/` — Vendored Superset monorepo (will become a git submodule). Contains the full upstream codebase: apps, packages, Docker configs, etc.
- `backend/` — Native Rust backend and multi-call binary (`superset-cli`). Handles the API server, database access, and is being extended with CLI commands ported from the TS CLI.
- `cli/` — Extracted TypeScript CLI source (`@superset/cli` + `@superset/cli-framework`). This is the reference implementation for CLI commands that are being ported to Rust, and can also be compiled to a single binary via Bun's `bun build --compile`.

## Technology Stack
| Layer | Technology |
|---|---|
| Backend Server | Rust, Axum, SQLx, PostgreSQL |
| CLI (Rust port) | Rust, Clap, Reqwest |
| CLI (TS reference) | TypeScript, Bun, tRPC client |
| Upstream Vendor | Bun monorepo, Turborepo, Next.js, Drizzle ORM |

## Git Conventions
- **No `Co-Authored-By` trailers** in commit messages.
- **No PII** (names, emails, internal URLs) in committed code, docs, or comments.
- Commit messages follow Conventional Commits: `feat:`, `fix:`, `test:`, `docs:`, `style:`, `chore:`.

## Code Quality Rules
- **Testing**: Target great test coverage across all actively developed code. Use `cargo llvm-cov` for the Rust backend to verify code paths are thoroughly exercised by tests.
- **Linting (Rust)**: Keep the codebase `clippy` clean. Always resolve warnings produced by `cargo clippy` before considering a task complete.
- **Linting (TypeScript)**: Use Biome for formatting and linting in the `cli/` directory. Run `bunx biome check --write` to auto-fix.
- Avoid `.unwrap()` and `panic!` in Rust production paths; prefer `Result` with `anyhow`/`thiserror`.

## Documentation and Plans
- Implementation and architecture porting plans go inside `backend/plans/`.
- OpenSpec is used for tracking changes. Artifacts are located in `backend/openspec/`.

## CLI Architecture
The Rust binary (`superset-cli`) uses a multi-call pattern:
- **No arguments** or `serve` → starts the Axum API backend server.
- **Subcommands** (e.g., `auth login`, `start`, `stop`) → executes CLI logic without starting the server.
- Configuration is read from and written to `~/.superset/config.json`.
- Host daemon manifests are stored per-organization at `~/.superset/host/<orgId>/manifest.json`.
