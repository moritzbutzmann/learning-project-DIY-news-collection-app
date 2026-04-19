# CLAUDE.md

## Runtime environment

This is a demo environment running inside a Docker container.

- **Ports**: Any application must bind to a port in the range `4200-4250`. Only these ports are forwarded by the container.
- **Host**: Applications must bind to `0.0.0.0` (not `localhost`/`127.0.0.1`) so they are reachable from outside the container.

## Installing software

The container runs a **fixed Docker image**. System-level software cannot be installed (no `apt install`, no new binaries, no system packages). Only **Python packages** (via `pip`) and **npm packages** can be added. If a task seems to require anything else, stop and tell the user — don't try to work around it.

## Transient filesystem

This container is **transient**. On every restart, anything outside this project directory is wiped — installed packages, system settings, global config, apt packages, etc. Only files inside the project directory survive.

- Any change made outside the project directory (system settings, globally installed tools, environment tweaks) must be **recorded in this project** (e.g. a setup script or notes file) so it can be reapplied after a restart.
- Assume a clean system on every session start.

## Installed packages

Every installed package — regardless of language or ecosystem — must be recorded in the appropriate manifest file inside the project directory. Packages not recorded there will be lost on container restart and cannot be easily recovered.

- **Python** (`pip`): install with
  ```
  pip install --no-cache-dir --break-system-packages <package>
  ```
  and record in `requirements.txt` or `pyproject.toml`.
- **JavaScript / Node** (`npm`, `yarn`, `pnpm`): install with the appropriate `--save` / `--save-dev` flags so the package is recorded in `package.json` (and the lockfile is updated).
- **Other ecosystems**: use the ecosystem's standard manifest (e.g. `Gemfile`, `go.mod`, `Cargo.toml`) and make sure every installed dependency is listed there before ending the session.