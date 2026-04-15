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

## Python packages

- Always install Python packages with:
  ```
  pip install --no-cache-dir --break-system-packages <package>
  ```
- Every installed package must be recorded in `requirements.txt` or `pyproject.toml` in the project directory. Packages not recorded there will be lost on container restart and cannot be easily recovered.
