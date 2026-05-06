# Hermes Agent — Cloud Deployment Guide

Hermes is packaged as a Docker image. The canonical way to run it persistently in the cloud is the **gateway** mode, which connects to Telegram (and optionally Discord, Slack, etc.) and keeps running indefinitely.

All configuration is done through environment variables and a persistent volume mounted at `/opt/data`. The container is fully stateless; state lives on the disk.

---

## Architecture at a Glance

| Component | Description |
|-----------|-------------|
| `hermes gateway run` | Main persistent process — runs all messaging gateways |
| `/opt/data` | Persistent volume: `.env`, `config.yaml`, sessions, memories, skills |
| Port `8642` | Built-in OpenAI-compatible API server + `/health` endpoint |
| `TELEGRAM_BOT_TOKEN` | Activates the Telegram gateway in long-polling mode |
| `TELEGRAM_WEBHOOK_URL` | Optional: switches Telegram to webhook mode |
| `OPENROUTER_API_KEY` (or other LLM key) | Required for the agent to think |

On first start the entrypoint script auto-bootstraps `/opt/data` (copies default `config.yaml` and `.env` from bundled examples) — no interactive setup wizard is needed in cloud deployments.

---

## Required Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | **Yes** (for Telegram) | Bot token from @BotFather |
| `OPENROUTER_API_KEY` | **Yes** (or one LLM key) | LLM provider key — OpenRouter gives access to 100+ models via one key |
| `ANTHROPIC_API_KEY` | Alt LLM | Direct Anthropic Claude API |
| `OPENAI_API_KEY` | Alt LLM | Direct OpenAI API |
| `GOOGLE_API_KEY` | Alt LLM | Google AI Studio / Gemini |
| `API_SERVER_KEY` | **Yes** (for web service) | Auth key for the built-in API server; generate with `openssl rand -hex 32` |
| `API_SERVER_ENABLED` | **Yes** (for web service) | Set to `"true"` to activate API server + health endpoint |
| `API_SERVER_HOST` | **Yes** (for web service) | Set to `"0.0.0.0"` to bind to all interfaces |
| `HERMES_ACCEPT_HOOKS` | **Yes** | Set to `"1"` — suppresses TTY prompts in non-interactive mode |
| `TELEGRAM_ALLOWED_USERS` | Recommended | Comma-separated Telegram user IDs allowed to chat; leave unset for open access |
| `TELEGRAM_HOME_CHANNEL` | Optional | Default Telegram chat ID for cron job delivery |
| `TELEGRAM_WEBHOOK_URL` | Optional | Switches from polling to webhook mode, e.g. `https://myapp.onrender.com/telegram` |
| `TELEGRAM_WEBHOOK_SECRET` | Optional | Validates incoming webhook requests from Telegram |
| `EXA_API_KEY` | Optional | Web search via Exa |
| `FIRECRAWL_API_KEY` | Optional | Web scraping via Firecrawl |
| `FAL_KEY` | Optional | Image generation via fal.ai |
| `GROQ_API_KEY` | Optional | Free-tier cloud voice transcription (Whisper) |

---

## Telegram Bot Setup

1. Open Telegram, search for **@BotFather**, start a chat.
2. Send `/newbot` and follow the prompts to create a bot.
3. Copy the **token** (looks like `7123456789:AAFxxxxxxxx`).
4. Find your **numeric user ID** using [@userinfobot](https://t.me/userinfobot).
5. Set `TELEGRAM_BOT_TOKEN=<token>` and `TELEGRAM_ALLOWED_USERS=<your-id>` in your cloud platform's environment variables.

> The bot starts in long-polling mode by default — no webhook URL is needed. Polling works on all platforms (Render, Railway, Fly.io, VPS, etc.).

---

## Render Deployment

### Prerequisites
- A Render account at [render.com](https://render.com)
- Repo pushed to GitHub or GitLab

### Steps

1. **Fork / push** the `hermes-agent` repository to your GitHub account.

2. In the Render dashboard, click **"New" → "Blueprint"** and connect your repository.
   Render will detect the `render.yaml` at the repo root.

3. Render will show the `hermes-gateway` service. Before applying, set the **secret** env vars in the UI (the ones marked `sync: false` in `render.yaml`):
   - `TELEGRAM_BOT_TOKEN`
   - `OPENROUTER_API_KEY` (or your preferred LLM key)
   - `API_SERVER_KEY` — generate with `openssl rand -hex 32`
   - `TELEGRAM_ALLOWED_USERS` (optional but recommended)

4. Click **"Apply"**. Render builds the Docker image and starts the service.

5. Watch the deploy logs — you should see:
   ```
   Starting hermes gateway ...
   Telegram: polling started
   API server listening on 0.0.0.0:8642
   ```

6. The Render health check (`GET /health` on port 8642) will turn green once the API server is ready.

7. Send a message to your Telegram bot to verify it responds.

### Enabling Webhook Mode (optional, lower latency)

Once the service is live, enable webhook mode for slightly lower response latency:

1. Note your Render service URL: `https://<service-name>.onrender.com`
2. In Render, add these env vars:
   ```
   TELEGRAM_WEBHOOK_URL = https://<service-name>.onrender.com/telegram
   TELEGRAM_WEBHOOK_SECRET = <generate with openssl rand -hex 32>
   ```
3. Trigger a redeploy. The gateway will automatically register the webhook with Telegram on startup.

### Persistent Disk

The `render.yaml` declares a 10 GB managed disk at `/opt/data`. This survives deploys and rollbacks. To resize, update `sizeGB` in `render.yaml` and re-apply the blueprint.

### CLI / Shell Access

To run a one-off CLI command against the running container:

1. In the Render dashboard, open the service → **Shell** tab.
2. The Hermes venv is activated by the entrypoint. Run:
   ```bash
   hermes chat -q "what is 2+2"         # one-shot query
   hermes sessions browse                # browse past sessions
   hermes cron list                      # list scheduled jobs
   hermes status                         # show gateway and tool status
   ```

---

## Railway Deployment

Railway supports Docker deployments without a config file — it auto-detects the Dockerfile.

### Steps

1. Create a new project at [railway.app](https://railway.app).
2. Click **"New Service" → "GitHub Repo"** → select your fork.
3. Railway detects the Dockerfile and builds automatically.
4. Under **"Service" → "Settings"**:
   - Set **Start Command**: `gateway run`
   - Set **Port**: `8642` (optional — only needed if you want the API server accessible)
5. Under **"Variables"**, add:
   ```
   TELEGRAM_BOT_TOKEN=<your-token>
   OPENROUTER_API_KEY=<your-key>
   API_SERVER_ENABLED=true
   API_SERVER_HOST=0.0.0.0
   API_SERVER_KEY=<random-key>
   HERMES_ACCEPT_HOOKS=1
   TELEGRAM_ALLOWED_USERS=<your-id>
   ```
6. Under **"Service" → "Volumes"**, attach a persistent volume to `/opt/data`.
7. Deploy. The service starts the gateway in long-polling mode.

---

## Docker Build and Run (Local / VPS)

### Build

```bash
cd hermes-agent
docker build -t hermes-agent:local .
```

### Run — Telegram gateway (long-polling)

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  -e TELEGRAM_BOT_TOKEN=your_token_here \
  -e OPENROUTER_API_KEY=your_key_here \
  -e API_SERVER_ENABLED=true \
  -e API_SERVER_HOST=0.0.0.0 \
  -e API_SERVER_KEY=your_api_server_key \
  -e HERMES_ACCEPT_HOOKS=1 \
  -e TELEGRAM_ALLOWED_USERS=your_telegram_id \
  hermes-agent:local gateway run
```

### Run — interactive CLI (against the same data directory)

```bash
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  hermes-agent:local
```

### Docker Compose (gateway + dashboard)

```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    command: gateway run
    ports:
      - "8642:8642"
      - "9119:9119"   # dashboard (only active when HERMES_DASHBOARD=1)
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - OPENROUTER_API_KEY=${OPENROUTER_API_KEY}
      - API_SERVER_ENABLED=true
      - API_SERVER_HOST=0.0.0.0
      - API_SERVER_KEY=${API_SERVER_KEY}
      - HERMES_ACCEPT_HOOKS=1
      - TELEGRAM_ALLOWED_USERS=${TELEGRAM_ALLOWED_USERS}
      - HERMES_DASHBOARD=1
```

Put your secrets in a `.env` file next to the compose file and run:
```bash
docker compose up -d
docker compose logs -f
```

---

## Health Checks

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Returns `{"status": "ok"}` — used by Render, load balancers |
| `GET /health/detailed` | Rich status for cross-container dashboard probing |

Health checks require `API_SERVER_ENABLED=true` and `API_SERVER_HOST=0.0.0.0`.

To test manually:
```bash
curl -H "Authorization: Bearer $API_SERVER_KEY" http://localhost:8642/health
```

---

## Render Service Verification Checklist

After deploying, verify:

- [ ] Deploy logs show `Telegram: polling started` (or `webhook registered` for webhook mode)
- [ ] Render health check is green (`GET /health` returns 200)
- [ ] Send `/start` to your Telegram bot — it should respond
- [ ] `docker logs hermes` (or Render log stream) shows agent activity on message receipt
- [ ] Persistent disk is mounted: Render dashboard shows disk usage > 0 after first start
- [ ] Sessions are saved: `ls /opt/data/sessions/` inside the container shell

---

## Troubleshooting

### Container exits immediately on startup
Check logs. Common causes:
- Missing `TELEGRAM_BOT_TOKEN` — the gateway fails to connect
- Missing LLM key — verify at least one of `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` is set

### Render health check fails (service shows unhealthy)
Verify these env vars are set:
```
API_SERVER_ENABLED=true
API_SERVER_HOST=0.0.0.0
API_SERVER_KEY=<non-empty value>
```

### Gateway starts but bot doesn't respond
- Check `TELEGRAM_ALLOWED_USERS` — if set, your user ID must be in the list
- Check logs for Telegram API errors

### "hooks_auto_accept" TTY prompt blocks startup
Set `HERMES_ACCEPT_HOOKS=1` (already included in `render.yaml`).

### Memory / OOM kills
Hermes + Playwright needs ~2 GB RAM. Use Render "standard" plan or above. If you don't use browser tools, "starter" (512 MB) may be sufficient for pure LLM chat.

### Data not persisting across deploys
Confirm the Render disk is attached (`/opt/data` mounted). In Render dashboard: Service → Disks tab. The disk must be created before the first deploy.
