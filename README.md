# Superset Linux Installer Build

> **⚠️ License Modification Notice**  
> This repository contains a modified version of the original Superset software. Specifically, the source code within the `upstream/superset/` submodule has been altered from its original form. These modifications introduce a custom offline backend (`packages/cli-wrapper`) and adjust internal Desktop code to run entirely locally, bypassing the original proprietary cloud infrastructure.  
> These changes are distributed in accordance with the [Elastic License 2.0 (ELv2)](upstream/superset/LICENSE.md). This project is intended for local use and is **not** provided as a hosted/managed cloud service.

[**Superset**](https://github.com/superset-sh/superset) is the code editor for AI agents. It orchestrates swarms of CLI-based coding agents (like Claude Code, Codex, and Cursor) in parallel, completely isolating them in their own Git worktrees to prevent interference. 

The upstream Superset application natively provides an `.AppImage` for Linux. This repository is a wrapper specifically designed to build **native `.deb` installers** for Debian-flavored distributions (like Ubuntu, Debian, Pop!_OS, etc).

We build and provide **two different flavors** of the `.deb` installer on our [GitHub Releases](https://github.com/oonid/superset-sh-dev/releases) page:

## 1. The Official Cloud Version (`superset-*.deb`)
This version is functionally **identical** to the official upstream `AppImage`, but packaged cleanly as a native Debian package.
* **How it works:** It connects directly to the official `superset.sh` cloud servers (API, Web, and ElectricSQL relay).
* **Benefits:** Zero setup required. Just install it, click "Log in with GitHub", and you're immediately placed on the official Superset free plan.
* **Use this if:** You just want to use Superset easily without managing your own servers and database.

## 2. The Local Dev Version (`superset-dev-*.deb`)
This version is our custom, heavily-modified fork designed to run **completely offline and privately**. It intercepts all cloud traffic and routes it to an embedded mock backend daemon (`packages/cli-wrapper`).
* **How it works:** It requires you to run local infrastructure (PostgreSQL via Docker) and launch the bundled backend daemon alongside the app.
* **Benefits:** 100% private data. No cloud limits. Complete control over your database.
* **Use this if:** You are developing Superset locally, or you require strict offline data isolation.

---

## Installation & Usage: Official Cloud Version

1. Download the `superset-<version>-linux-amd64.deb` file from the Releases page.
2. Install it via the terminal:
   ```bash
   sudo dpkg -i superset-1.13.0-linux-amd64.deb
   sudo apt-get install -f # Resolves any missing system dependencies (like libnss3)
   ```
3. Launch the main application from your OS application menu, or run `superset` in your terminal!

---

## Installation & Usage: Local Dev Version

### 1. Start the Local Infrastructure (Docker)
Because the `superset-dev` flavor relies on a local backend instead of the official cloud, you must run the required backing infrastructure (PostgreSQL, Neon Proxy, and ElectricSQL). 

A complete `docker-compose.yml` is provided in the `upstream/superset` directory.
```bash
cd upstream/superset
docker compose up -d
```
This automatically provisions your database (`:5432`), the Neon HTTP proxy (`:4444`), and ElectricSQL (`:3100`).

### 2. Install the Package
Download the `superset-dev-<version>-linux-amd64.deb` file from the Releases page.
```bash
sudo dpkg -i superset-dev-1.13.0-linux-amd64.deb
sudo apt-get install -f
```

### 3. Use the Companion CLI (Backend Daemon)
The custom TypeScript CLI / Backend daemon is natively bundled directly inside the Linux installation directory. To start the local proxy servers and mock backend, invoke the embedded binary and pass in your environment variables:

```bash
DATABASE_URL="postgres://postgres:postgres@db.localtest.me:4444/main" \
GITHUB_APP_ID="xxxxx" \
GITHUB_APP_PRIVATE_KEY="$(cat superset-sh-dev.privatekey.pem)" \
/opt/superset-dev/resources/resources/bin/superset-dev serve --web-port 3000 --api-port 3001
```
*Tip: You may want to create a symlink or an alias in your `~/.bashrc` for easier access to the CLI!*

### 4. Launch the Desktop App
With the daemon running, open a new terminal and run:
```bash
superset-dev
```

---

## Architecture & Responsibility Breakdown (Local Dev Version)

When using the `superset-dev` flavor, the responsibilities are split between the graphical Desktop app and the custom `serve` daemon.

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
- **Version Poller:** Periodically queries GitHub releases to provide native update notifications mirroring the official AppImage behavior.

### 🔌 Implemented Mock tRPC Endpoints
The backend daemon dynamically intercepts and implements the following tRPC endpoints to simulate the cloud:
- **User & Auth:** `user.me`, `user.completeOnboarding`, `organization.list`
- **Devices & Hosts:** `device.registerDevice`, `host.ensure`, `v2Host.rename`
- **Projects:** `v2Project.create`, `v2Project.get`, `v2Project.list`, `v2Project.update`, `v2Project.delete`, `v2Project.findByGitHubRemote`, `v2Project.linkRepoCloneUrl`
- **Workspaces:** `v2Workspace.create`, `v2Workspace.update`, `v2Workspace.list`, `v2Workspace.delete` / `v2Workspace.deleteMainForHost`, `v2Workspace.getFromHost`, `v2Workspace.listFromHost`
- **Integrations:** `integration.github.getInstallation`, `git.getStatus`, `chat.getModels`
- **Operations:** `workspaceCleanup.inspect`, `workspaceCleanup.destroy`

## Known Linux Packaging Quirks
**Redundant SQLite Migrations Path:** The upstream codebase expects Electron database migrations to be located at `process.resourcesPath + "/resources/migrations"`. During the `.deb` compilation, this technically points to a nested `resources/resources/` directory structure. For now, this is left untouched in the upstream code to avoid merge conflicts.
