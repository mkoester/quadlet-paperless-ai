# quadlet-paperless-ai

Quadlet setup for [Paperless AI](https://github.com/clusterzx/paperless-ai) — an AI companion for [Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx) that automatically analyses, tags, and summarises your documents (`docker.io/clusterzx/paperless-ai`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `paperless-ai.container` | Quadlet unit file |
| `paperless-ai.env` | Default environment variables |
| `paperless-ai.override.env.template` | Template for local overrides (API credentials, AI provider) |
| `paperless-ai-backup.service` | Systemd service: rsync data directory to backup staging |
| `paperless-ai-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/paperless-ai -s /usr/sbin/nologin paperless-ai

REPO_URL=https://github.com/mkoester/quadlet-paperless-ai.git
REPO=~paperless-ai/quadlet-paperless-ai
```

```sh
# 2. Enable linger
sudo loginctl enable-linger paperless-ai

# 3. Clone this repo into the service user's home
sudo -u paperless-ai git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u paperless-ai mkdir -p ~paperless-ai/.config/containers/systemd
sudo -u paperless-ai mkdir -p ~paperless-ai/data

# 5. Create .override.env from template and fill in required values
sudo -u paperless-ai cp $REPO/paperless-ai.override.env.template $REPO/paperless-ai.override.env
sudo -u paperless-ai nano $REPO/paperless-ai.override.env

# 6. Symlink all quadlet files from the repo
sudo -u paperless-ai ln -s $REPO/paperless-ai.container ~paperless-ai/.config/containers/systemd/paperless-ai.container
sudo -u paperless-ai ln -s $REPO/paperless-ai.env ~paperless-ai/.config/containers/systemd/paperless-ai.env
sudo -u paperless-ai ln -s $REPO/paperless-ai.override.env ~paperless-ai/.config/containers/systemd/paperless-ai.override.env

# 7. Reload systemd (generates the unit from the .container file)
sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user daemon-reload

# 8. Start manually (on demand)
sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user start paperless-ai

# 9. Verify
sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user status paperless-ai
```

> **On-demand use:** The `.container` file has no `[Install]` section, so the quadlet generator will not auto-start it at boot. Just `start` and `stop` it as needed.

## Configuration

### Environment variables

`paperless-ai.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |
| `PAPERLESS_AI_PORT` | `3000` | HTTP port inside the container |
| `SCAN_INTERVAL` | `*/30 * * * *` | Cron schedule for scanning new documents |
| `ADD_AI_PROCESSED_TAG` | `yes` | Add a tag after AI processing |
| `AI_PROCESSED_TAG_NAME` | `ai-processed` | The tag name to add |

`paperless-ai.override.env` (created from template) must set:

| Variable | Description |
|---|---|
| `PAPERLESS_API_URL` | Full URL to the Paperless-ngx API, e.g. `http://192.168.1.10:8000/api` |
| `PAPERLESS_API_TOKEN` | API token from Paperless-ngx (Settings → API → Generate token) |
| `PAPERLESS_USERNAME` | Your Paperless-ngx username |
| `AI_PROVIDER` | `ollama`, `openai`, or `custom` |
| `OLLAMA_API_URL` | Ollama endpoint (if `AI_PROVIDER=ollama`) |
| `OLLAMA_MODEL` | Ollama model name (if `AI_PROVIDER=ollama`) |
| `OPENAI_API_KEY` | OpenAI API key (if `AI_PROVIDER=openai`) |
| `OPENAI_MODEL` | OpenAI model name (if `AI_PROVIDER=openai`) |

To apply changes after editing the override file:

```sh
sudo -u paperless-ai nano ~paperless-ai/quadlet-paperless-ai/paperless-ai.override.env
sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user restart paperless-ai
```

## Ollama (local AI, GPU-accelerated)

If you are running Ollama on the host, set `OLLAMA_API_URL=http://host.containers.internal:11434` in the override file. This special hostname resolves to the host from inside the container (rootless Podman adds it automatically).

Recommended models for document analysis (16 GB VRAM):

| Model | Pull command | Notes |
|---|---|---|
| `llama3.2` | `ollama pull llama3.2` | Good baseline, fast |
| `mistral` | `ollama pull mistral` | Strong at structured extraction |
| `llama3.1:8b` | `ollama pull llama3.1:8b` | Good quality, fits comfortably |

## Web UI

Once started, the Paperless AI web UI is available at http://localhost:3000.

On first start, navigate there to complete the initial setup wizard (connect to Paperless-ngx, configure the AI provider, and optionally set a system prompt).

## Reverse proxy (Caddy)

If you expose the UI remotely, add a site block to your Caddyfile:

```
paperless-ai.example.com {
    reverse_proxy localhost:3000
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Paperless AI stores its state in `/app/data` inside the container (`~paperless-ai/data/` on the host): the SQLite database with document processing history, tags, correspondents, and configuration. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup.

```sh
# 1. Create backup staging directory (owned by paperless-ai, readable by backup-readers group)
sudo mkdir -p /var/backups/paperless-ai/data
sudo chown -R paperless-ai:backup-readers /var/backups/paperless-ai
sudo chmod -R 750 /var/backups/paperless-ai

# 2. Symlink the backup service and timer from the repo
sudo -u paperless-ai mkdir -p ~paperless-ai/.config/systemd/user
sudo -u paperless-ai ln -s $REPO/paperless-ai-backup.service ~paperless-ai/.config/systemd/user/paperless-ai-backup.service
sudo -u paperless-ai ln -s $REPO/paperless-ai-backup.timer ~paperless-ai/.config/systemd/user/paperless-ai-backup.timer

# 3. Enable and start the timer
sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user daemon-reload
sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user enable --now paperless-ai-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@paperless-ai-host:/var/backups/paperless-ai/ /path/to/local/backup/paperless-ai/
```

## Notes

- Port `3000` is bound to `127.0.0.1` only — use a reverse proxy or SSH tunnel to access it remotely.
- All persistent state is stored at `~paperless-ai/data/` on the host.
- The container runs as root internally; rootless Podman maps that to the `paperless-ai` service user on the host.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning)):
  ```sh
  sudo -u paperless-ai XDG_RUNTIME_DIR=/run/user/$(id -u paperless-ai) systemctl --user enable --now podman-image-prune@30.timer
  ```
