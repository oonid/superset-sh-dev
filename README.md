# Superset Linux Installer Build

The upstream Superset application currently only natively targets macOS. This repository is a wrapper specifically designed to build a custom Linux installer (`.deb`) for Debian-flavored distributions.

### Requirements for Linux Users
To run Superset on Debian/Ubuntu, you will need:
1. **The `.deb` Installer**: Download the latest release from the [GitHub Releases](https://github.com/oonid/superset-sh-dev/releases) page. (Automatically built via GitHub Actions!)
2. **PostgreSQL**: A running local PostgreSQL instance for the backend database.

*Note: The unified TypeScript CLI is natively bundled **inside** the `.deb` installer, so you do not need to download the CLI separately!*

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

## Architecture
- `upstream/superset/`: The upstream Superset monorepo (submodule).

## Features Implemented (`cli-wrapper`)
Our custom local TypeScript backend provides the following features out of the box:
- **Unified Hono API Server**: Bundled cleanly alongside the companion CLI.
- **Dual-Port Daemon**: Listens dynamically for API (`3001`) and Web proxy traffic (`3000`) simultaneously via the `serve` command.
- **Local Authentication Bypass**: Automatic dev login exposure in the UI + disabled auth constraints for the local daemon.
- **Secure Origin Interceptor**: Patched into the Desktop App's main process to allow local CORS traffic natively.
- **GitHub App Integration**: Full port of the GitHub installation flow! Includes generating secure `state` JWTs, handling local callbacks (`/api/github/callback`), mapping to the correct session User ID, and pushing installation metadata straight into the local database.

## Known Linux Packaging Quirks
**Redundant SQLite Migrations Path:** The upstream codebase expects Electron database migrations to be located at `process.resourcesPath + "/resources/migrations"`. During the `.deb` compilation, this technically points to a nested `resources/resources/` directory structure. For now, this is left untouched in the upstream code to avoid merge conflicts.
