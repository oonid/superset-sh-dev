# Superset Linux Installer Build

[**Superset**](https://github.com/superset-sh/superset) is the code editor for AI agents. It orchestrates swarms of CLI-based coding agents (like Claude Code, Codex, and Cursor) in parallel, completely isolating them in their own Git worktrees to prevent interference. 

The upstream Superset application currently only natively targets macOS. This repository is a wrapper specifically designed to build a custom Linux installer (`.deb`) for Debian-flavored distributions.

### Requirements for Linux Users
To run Superset on Debian/Ubuntu, you will need:
1. **The `.deb` Installer**: Download the latest release from the [GitHub Releases](https://github.com/oonid/superset-sh-dev/releases) page. (Automatically built via GitHub Actions!)
2. **PostgreSQL**: A running local PostgreSQL instance for the backend database.

*Note: The unified TypeScript CLI is natively bundled **inside** the `.deb` installer, so you do not need to download the CLI separately!*

## Local Backend Services (Docker)
Because the Linux wrapper relies on a local backend instead of the official cloud services, you must run the required backing infrastructure (PostgreSQL, Neon Proxy, and ElectricSQL). A complete `docker-compose.yml` is provided in the `upstream/superset` directory for this exact purpose.

To spin up your local infrastructure using Docker:
```bash
cd upstream/superset
docker compose up -d
```
This will automatically provision your database (`:5432`), the Neon HTTP proxy (`:4444`), and ElectricSQL (`:3100`), perfectly matching the default environment variables expected by the `.deb` application.

## Installation & Usage

### 1. Install the Package
Once you have downloaded the `.deb` installer from the Releases page, install it via the terminal:
```bash
sudo dpkg -i superset-dev-1.12.5-amd64.deb
sudo apt-get install -f # To resolve any missing system dependencies
```

### 2. Launch the Desktop App
You can launch the main graphical Desktop application from your OS application menu, or directly from the terminal by typing:
```bash
superset-dev
```

### 3. Use the Companion CLI (Backend Daemon)
The custom TypeScript CLI / Backend daemon is natively bundled directly inside the Linux installation directory. To use the CLI (for example, to start the local proxy servers), invoke the embedded binary and pass in your environment variables.

Here is a complete example to start the offline daemon:
```bash
DATABASE_URL="postgres://postgres:postgres@db.localtest.me:4444/main" \
GITHUB_APP_ID="xxxxx" \
GITHUB_APP_PRIVATE_KEY="$(cat superset-sh-dev.privatekey.pem)" \
/opt/superset-dev/resources/resources/bin/superset-dev serve --web-port 3000 --api-port 3001
```
*Tip: You may want to create a symlink or an alias in your `~/.bashrc` for easier access to the CLI!*

## Architecture & Responsibility Breakdown
- `upstream/superset/`: The upstream Superset monorepo (submodule).
- `packages/cli-wrapper/`: Our custom TypeScript CLI layer that powers the local backend.

Because this Linux build is designed to run **completely offline** (without the official Superset cloud services), the responsibilities are split between the graphical Desktop app and the custom `serve` daemon.

### 🖥️ The Superset Desktop App (Frontend)
The GUI acts purely as the presentation and orchestration layer:
- **Agent UI:** Renders the chat interface for interacting with Claude Code, Codex, and Cursor.
- **Terminal Management:** Spins up and visualizes the embedded pty (pseudo-terminals) for agents.
- **Git Worktree Isolation:** Visually separates the different agent worktrees in the UI.
- **Local SQLite Cache:** Maintains a fast, local SQLite database for UI caching and ElectricSQL sync state.
- **IPC Communication:** Handles Electron-level system access (file system reads, terminal execution).

### ⚙️ The Custom Local Backend (`serve` Daemon)
The custom embedded CLI completely replaces the official cloud infrastructure:
- **Unified Hono API Server (`:3001`):** Bundled cleanly alongside the companion CLI to serve TRPC requests.
- **Web Proxy & OAuth (`:3000`):** Listens dynamically for OAuth callbacks and Web proxy traffic simultaneously.
- **Local Authentication Bypass:** Replaces the rigid Better-Auth cloud verification with local JWT minting and exposes the "Sign in as Local Admin (dev)" button.
- **PostgreSQL Source of Truth:** Connects directly to your local Postgres instance instead of Drizzle Cloud, managing organizations, projects, and users.
- **Secure Origin Interceptor:** Patched into the Desktop App's main process to allow local CORS traffic natively.
- **GitHub App Integration:** Full port of the GitHub installation flow! Includes generating secure `state` JWTs, handling local callbacks (`/api/github/callback`), mapping to the correct session User ID, and pushing installation metadata straight into the local database.
- **ElectricSQL Disconnect Guard:** Prevents the desktop from infinite-looping when trying to sync with the missing cloud ElectricSQL server.

## Known Linux Packaging Quirks
**Redundant SQLite Migrations Path:** The upstream codebase expects Electron database migrations to be located at `process.resourcesPath + "/resources/migrations"`. During the `.deb` compilation, this technically points to a nested `resources/resources/` directory structure. For now, this is left untouched in the upstream code to avoid merge conflicts.
