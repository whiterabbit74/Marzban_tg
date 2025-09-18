<p align="center">
  <img src="https://github.com/Gozargah/Marzban-docs/raw/master/screenshots/logo-light.png" width="160" height="160" alt="Marzban logo"/>
</p>

<h1 align="center">Marzban – Bot First Edition</h1>

<p align="center">
  A fork of <a href="https://github.com/Gozargah/Marzban">Marzban</a> focused on the REST API, Telegram bot, and CLI – without the React dashboard.
</p>

---

## Contents

- [Overview](#overview)
- [What Changed in This Fork](#what-changed-in-this-fork)
- [Key Features](#key-features)
- [Quick Start (Docker)](#quick-start-docker)
- [Configuring Xray Inbounds](#configuring-xray-inbounds)
- [Environment Variables](#environment-variables)
- [Telegram Bot](#telegram-bot)
- [Command Line Interface](#command-line-interface)
- [HTTP API](#http-api)
- [Operations and Maintenance](#operations-and-maintenance)
- [Contributing](#contributing)
- [License](#license)

## Overview

Marzban is a Python/FastAPI control plane for [Xray-core](https://github.com/XTLS/Xray-core). It provisions users, keeps xray in sync, exposes subscription links, and automates notifications. This fork trims the web dashboard and ships a bot-first workflow suited for minimal servers or headless deployments.

## What Changed in This Fork

- Removed the React dashboard and static assets – everything is managed through the API, the Telegram bot, or the CLI.
- Simplified CI pipelines; no Node toolchain is required.
- Documentation is rewritten for bot/API usage and highlights how to run without a web front-end.

## Key Features

- ⚙️ REST API with OpenAPI schema (`/docs`, `/openapi.json`).
- 🤖 Built-in Telegram bot for provisioning, usage reports, and alerts.
- 🧰 `marzban-cli` for scripted workflows and local administration.
- 🔁 Full support for VMess, VLESS, Trojan, and Shadowsocks – limited only by your Xray config.
- 📡 Multi-node architecture via [marzban-node](https://github.com/gozargah/marzban-node).
- 📊 User quotas, expiry timers, recurring resets, custom templates, and webhook notifications.

## Quick Start (Docker)

> Requirements: Docker 24+, git, and a Telegram bot token (create one with [@BotFather](https://t.me/botfather)).

1. **Clone and build**
   ```bash
   git clone https://github.com/whiterabbit74/Marzban_tg.git marzban-bot
   cd marzban-bot
   docker build -t marzban-bot:latest .
   ```

2. **Create an environment file**
   ```bash
   cat >.env <<'ENV'
   # Core server
   SQLALCHEMY_DATABASE_URL=sqlite:////var/lib/marzban/db.sqlite3
   UVICORN_HOST=0.0.0.0
   UVICORN_PORT=8000

   # Telegram bot
   TELEGRAM_API_TOKEN=123456789:example-token
   TELEGRAM_ADMIN_ID=111111111

   # Optional hardening
   ALLOWED_ORIGINS=http://localhost
   ENV
   ```

3. **Select an inbound in `xray_config.json`**
   Edit the file shipped with the repo and keep only the protocols you need. A minimal VLESS listener on port `2999` looks like:
   ```json
   {
     "inbounds": [
       {
         "tag": "VLESS TCP",
         "listen": "0.0.0.0",
         "port": 2999,
         "protocol": "vless",
         "settings": {
           "decryption": "none",
           "clients": []
         },
         "streamSettings": {
           "network": "tcp",
           "security": "none",
           "tcpSettings": {
             "header": { "type": "none" }
           }
         }
       }
     ],
     "outbounds": [
       { "protocol": "freedom", "tag": "DIRECT" },
       { "protocol": "blackhole", "tag": "BLOCK" }
     ]
   }
   ```

4. **Run the container**
   ```bash
   docker run -d --name marzban \
     --restart always \
     --env-file .env \
     -v /var/lib/marzban:/var/lib/marzban \
     -p 2999:2999 \   # expose your Xray inbound(s)
     -p 127.0.0.1:8000:8000 \   # expose the API only on localhost
     marzban-bot:latest \
     bash -c 'alembic upgrade head && uvicorn main:app --host 0.0.0.0 --port 8000'
   ```

5. **Create the first admin**
   ```bash
   docker exec -e MARZBAN_ADMIN_PASSWORD=strongpass marzban \
     marzban-cli admin create --username admin --sudo --telegram-id 111111111
   ```

6. **Verify**
   ```bash
   docker logs -f marzban
   curl http://127.0.0.1:8000/docs
   docker exec marzban marzban-cli user list
   ```

## Configuring Xray Inbounds

Marzban reads `xray_config.json` at startup and builds user links based on the inbounds defined there. Keep the file minimal – one inbound per protocol you intend to hand out. For TLS, REALITY, or WebSocket deployments you can reuse upstream examples; Marzban forwards client settings to Xray unchanged.

When you modify `xray_config.json`, restart the container:
```bash
docker restart marzban
```

## Environment Variables

Marzban loads configuration from environment variables (or a `.env` file). Important categories:

### Core service

| Variable | Description |
| --- | --- |
| `SQLALCHEMY_DATABASE_URL` | Database connection string. SQLite is fine for single-node setups. |
| `UVICORN_HOST` / `UVICORN_PORT` / `UVICORN_UDS` | Bind address, TCP port, or UNIX socket for the API. |
| `UVICORN_SSL_CERTFILE` / `UVICORN_SSL_KEYFILE` / `UVICORN_SSL_CA_TYPE` | Serve HTTPS directly from Uvicorn (`UVICORN_SSL_CA_TYPE=private` allows self-signed certs). |
| `ALLOWED_ORIGINS` | Comma-separated list of CORS origins for API clients. |
| `DOCS` | Set to `true` to expose Swagger/Redoc. |

### Xray integration

| Variable | Description |
| --- | --- |
| `XRAY_JSON` | Path to the Xray config file (defaults to `./xray_config.json`). |
| `XRAY_EXECUTABLE_PATH` / `XRAY_ASSETS_PATH` | Paths to the Xray binary and geo assets inside the container/host. |
| `XRAY_SUBSCRIPTION_URL_PREFIX` | Optional prefix (e.g. `https://panel.example.com`). |
| `XRAY_SUBSCRIPTION_PATH` | URL segment for subscription links (default: `sub`). |
| `XRAY_EXCLUDE_INBOUND_TAGS` | Space-separated inbound tags to ignore when generating links. |

### Authentication & automation

| Variable | Description |
| --- | --- |
| `SUDO_USERNAME` / `SUDO_PASSWORD` | Seed a superuser on startup (alternative to `marzban-cli admin import-from-env`). |
| `TELEGRAM_API_TOKEN` | Bot token from @BotFather. |
| `TELEGRAM_ADMIN_ID` | Comma-separated Telegram user IDs allowed to administer via bot. |
| `TELEGRAM_PROXY_URL` | Optional proxy for Telegram traffic. |
| `WEBHOOK_ADDRESS` / `WEBHOOK_SECRET` | Comma-separated webhook targets and shared secret header. |
| `NOTIFY_*` variables | Fine-grained toggles for notification events (status change, quota reached, etc.). |

The full list (including subscription templates, Discord logging, job intervals, and advanced JSON options) remains in `config.py` and matches upstream defaults.

## Telegram Bot

Once the container starts, Marzban brings up the Telegram bot if `TELEGRAM_API_TOKEN` is present. Make sure the bot is unblocked and added to a private chat with each admin ID. Useful commands:

- `/start` – show the main menu and bot status.
- `/new_user` – create a user with your preferred protocol template.
- `/users` – list active accounts and quotas.
- `/backup_now` – trigger the backup service when configured.
- Inline buttons let you reset traffic, extend expiry, revoke links, and apply predefined plans.

Bot actions mirror the REST/CLI operations, so anything scripted is also available in chat.

## Command Line Interface

`marzban-cli` is available inside the container (or locally if you install the package). Common patterns:

```bash
# List users
docker exec marzban marzban-cli user list

# Generate a subscription link
docker exec marzban marzban-cli subscription get-link --username alice

# Export V2Ray configs (base64)
docker exec marzban marzban-cli subscription get-config \
  --username alice --format v2ray --base64
```

To use the CLI on the host:
```bash
python3 -m pip install -r requirements.txt
ln -s $(pwd)/marzban-cli.py ~/.local/bin/marzban-cli
marzban-cli completion install
```
Set `MARZBAN_URL`, `MARZBAN_USERNAME`, and `MARZBAN_PASSWORD` environment variables (or use the `--url` / `--auth` options) to point the CLI at a remote instance.

## HTTP API

FastAPI exposes comprehensive REST endpoints under `/api`. Highlights:

- `POST /api/user` – create a user (pass `UserCreate` JSON payload).
- `PUT /api/user/{username}` – modify quotas, protocols, notes.
- `POST /api/user/{username}/reset` – reset traffic usage.
- `GET /api/subscription/{token}` – retrieve subscription contents for clients.

Interactive documentation lives at `/docs` (Swagger UI) and `/redoc`. Add bearer authentication by calling `POST /api/admin/token` with admin credentials.

## Operations and Maintenance

- **Logs** – `docker logs -f marzban` shows API and validator output; Xray messages are prefixed with `Xray core`.
- **Backups** – run `marzban backup-service` (inside the container) to configure scheduled archive delivery via Telegram, or simply archive `/var/lib/marzban` plus `.env`.
- **Upgrades** – pull the latest code, rebuild the image, and restart:
  ```bash
  git pull origin master
  docker build -t marzban-bot:latest .
  docker stop marzban && docker rm marzban
  # re-run the docker command from the quick start
  ```
- **Health checks** – monitor `GET /api/system/health`, or integrate with the webhook system for automatic alerts.

## Contributing

Issues and pull requests are welcome. Please base feature branches on `master`, keep commits focused, and follow the style enforced by `autopep8 --max-line-length 120`. If you plan large changes, start a discussion in the issue tracker first.

## License

Marzban is distributed under the terms of the [GNU General Public License v3](LICENSE).

---

Need help or ideas? Join the upstream Telegram community at [@gozargah_marzban](https://t.me/gozargah_marzban).
