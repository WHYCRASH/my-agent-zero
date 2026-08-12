# Agent Zero — Ubuntu 24.04 Migration

Branch: `ubuntu` (pushed from laptop, fork `WHYCRASH/my-agent-zero`)

## Goal
Run Agent Zero on an updated Ubuntu 24.04 image instead of Kali rolling, with a defined package set, Playwright + headless Chromium, and UV. Lean image: no SearXNG, no LibreOffice/Xpra desktop stack by default; nginx→Caddy is an open TODO.

## Changed

### `DockerfileLocal` (root)
- `FROM ubuntu:24.04` (was `agent0ai/agent-zero-base:latest` Kali).
- One RUN block: `apt-get update && apt-get upgrade -y` + `--no-install-recommends` install of the requested packages, ending with `rm -rf /var/lib/apt/lists/*` (anti-bloat requirement).
- Requested: locales tzdata python3 python3-pip git curl ca-certificates build-essential ffmpeg supervisor nodejs npm tesseract-ocr tesseract-ocr-script-latn poppler-utils unzip p7zip-full openssh-server cron. Added `python3-venv`, `sudo`, `wget` (required by Ubuntu venv creation and helper scripts).
- Locale en_US.UTF-8, TZ UTC.
- Dual venv layout preserved: `/opt/venv` + `/opt/venv-a0`, both Python 3.12.3 (system interpreter; no pyenv compile). CPU torch==2.4.0/torchvision==0.19.0 in `/opt/venv-a0`.
- UV installed after the apt cleanup: `curl ... | UV_INSTALL_DIR=/usr/local/bin sh`.
- `EXPOSE 22 80 443`.

### `docker-compose.yml` (new, root)
- One service `agent-zero`, builds DockerfileLocal, image `agent-zero:ubuntu-24.04`, container `agentubuntu`.
- Ports 9000:80 (WebUI), 2222:22 (SSH), 4343:443 (TLS terminator). Volumes `./usr:/a0/usr`, `./tmp:/a0/tmp`. `tty: true`, `stdin_open: true`, `restart: unless-stopped`.

### Removed entirely
- SearXNG: `docker/base/fs/etc/searxng/`, `docker/base/fs/ins/install_searxng.sh`, `install_searxng2.sh`, `docker/run/fs/etc/searxng/`, `docker/run/fs/exe/run_searxng.sh`, supervisord `[program:run_searxng]`, base Dockerfile searxng step, SearXNG purpose line in `docker/base/AGENTS.md`.
- LibreOffice + Xpra installer: `docker/run/fs/ins/install_additional.sh` (also removed the `install_additional.sh` RUN from both root DockerfileLocal and `docker/run/Dockerfile`).
- nginx leftover: `docker/run/fs/etc/nginx/nginx.conf`.

### TODOs (documented, not done)
- Xpra desktop access — re-add later if needed, likely via a separate image/service.
- Replace nginx with Caddy — the stacked service currently terminates differently; Caddy container/service is future work.

## Working (verified on laptop image `agent-zero:ubuntu-24.04`)
- Ubuntu 24.04.4 LTS; Python 3.12.3 in both venvs; uv 0.12.3; Playwright 1.52.0 with headless Chromium at `/a0/tmp/playwright`.
- Binaries: ffmpeg, node, npm, tesseract, pdftoppm, pdfinfo, git, curl, sshd, cron, unzip, 7z, supervisord 4.2.5.
- `rm -rf /var/lib/apt/lists/*` requirement honored (no package lists remain).
- Container `agentubuntu` running, WebUI accessible and functional (user-verified 2026-08-12).

## Not working / gaps
- No SearXNG search backend: the old `tools/search_engine.py` used `helpers/searxng.py`, which is still present in code but requires a local SearXNG service; that search path is effectively broken without a service. (Search via Exa/hound in the live stack is unaffected.)
- `helpers/searxng.py` + `tools/search_engine.py` remain in the tree as untracked legacy; removing them is a follow-up (they currently reference a service that no longer ships).
- No nginx/Caddy inside the image; the WebUI runs on uvicorn port 80 (host 9000 mapping).
- `_office` plugin (LibreOffice) remains `always_enabled: false` and is unusable without the desktop stack — leave disabled unless Xpra stack returns.
