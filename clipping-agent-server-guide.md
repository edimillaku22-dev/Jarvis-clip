# Clipping Agent on a Linux server (operated from Android)

The whole system runs on a rented Linux server: dashboard, worker, clip production (OpenShorts + local AI) and posting.
You only use the **browser** on your Android device.

The server is **not reachable from the internet**. Only your own devices can reach it, through Tailscale (a private network,
free for personal use).

```
Android (browser + Tailscale app)  ──private network──►  Linux server
                                                          ├─ Dashboard (Next.js)    127.0.0.1:3000 → https://<server>.ts.net
                                                          ├─ Worker every 5 min (systemd)
                                                          ├─ OpenShorts (Docker)    127.0.0.1:8000
                                                          └─ Ollama (local AI)      qwen2.5:7b
```

## 0. What you need

- **Server:** Ubuntu 24.04 LTS, at least **8 vCPU, 16 GB RAM, 150 GB SSD**. Video processing runs on the CPU (no GPU needed).
  Fewer cores means slower clip production. Monthly price: compare providers.
- **Your own accounts:** TikTok/YouTube (+ developer apps), Vyro and/or Whop. Nothing from the original operator is included:
  no logins, campaigns or data.
- **Android:** the Tailscale app (Play Store) and an SSH app (e.g. Termux: `pkg install openssh`) for the setup commands.

## 1. Prepare the server (as root over SSH, once)

```bash
adduser clipper && usermod -aG sudo clipper
```
```bash
apt update && apt install -y git curl sqlite3 attr build-essential
```
```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | bash - && apt install -y nodejs
```
```bash
curl -fsSL https://get.docker.com | sh && usermod -aG docker clipper
```
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
```bash
curl -fsSL https://tailscale.com/install.sh | sh && tailscale up
```

Log in with the same Tailscale account you use on Android. Then work only as `clipper`: `su - clipper`.

Ollama has to be reachable from the Docker containers:

```bash
sudo systemctl edit ollama
```
Add this and save:
```
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```
```bash
sudo systemctl restart ollama && ollama pull qwen2.5:7b
```

**Firewall:** make sure port 11434 (and 3000/8000) are **not** open to the internet. With ufw:
`sudo ufw default deny incoming && sudo ufw allow in on tailscale0 && sudo ufw allow OpenSSH && sudo ufw enable`.

## 2. OpenShorts (clip production)

```bash
git clone https://github.com/mutonby/openshorts ~/openshorts
```

`~/openshorts/.env` (the tested settings from the original setup; no API keys needed):

```
LLM_BASE_URL=http://host.docker.internal:11434/v1
LLM_MODEL=qwen2.5:7b
LLM_SCORE_BATCH=3
LLM_TIMEOUT=900
WHISPER_MODEL=small
WHISPER_DEVICE=cpu
WHISPER_COMPUTE=int8
TRANSCRIBE_BACKEND=whisper
FFMPEG_ENCODER=x264
JOB_RETENTION_SECONDS=604800
MAX_FILE_SIZE_MB=8192
SCENE_ENGINE=pyscenedetect
CLIP_WORKERS=3
```

`SCENE_ENGINE` and `CLIP_WORKERS` are measured: a 30-minute segment took 11.6 instead of 30.1 minutes. Rule of thumb for
`CLIP_WORKERS`: about half the vCPUs.

`~/openshorts/docker-compose.override.yml` (ports only on 127.0.0.1, plus host access on Linux):

```yaml
services:
  backend:
    ports: !override
      - "127.0.0.1:8000:8000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
  frontend:
    ports: !override
      - "127.0.0.1:5175:5173"
  renderer:
    ports: !override
      - "127.0.0.1:3100:3100"
```

```bash
cd ~/openshorts && docker compose up -d --build
```

Test: `curl -s localhost:8000` should answer.

## 3. Install the Clipping Agent

Get the code (GitHub link or the `.bundle` file you received):

```bash
git clone <link-or-file.bundle> ~/clipping-agent && cd ~/clipping-agent
```
```bash
npm ci
```

Server address (once, from `tailscale status`, e.g. `clipper.tail1234.ts.net`). Expose the dashboard **only in the
private network** over HTTPS:

```bash
sudo tailscale serve --bg 3000
```

Tell the dashboard its address (for OAuth redirects). The services read `data/server.env`:

```bash
mkdir -p data && echo "PUBLIC_URL=https://clipper.tail1234.ts.net" > data/server.env
```

Start the background services (builds the dashboard, then dashboard + worker every 5 min + nightly check 03:30):

```bash
npm run agent -- install-systemd
```
```bash
sudo loginctl enable-linger clipper
```

Status: `systemctl --user status clipping-dashboard clipping-tick.timer`. Logs: `~/clipping-agent/data/logs/`.

**On Android:** Tailscale app on → open `https://clipper.tail1234.ts.net` in the browser. That's the dashboard.

## 4. Connect your accounts (in the dashboard → Connections)

The page shows the exact **redirect URIs** with your server address. Register them in each developer console:

| Platform | Console | Redirect URI |
|---|---|---|
| YouTube | console.cloud.google.com (OAuth client "Web") | `https://<server>.ts.net/api/oauth/youtube/callback` |
| TikTok | developers.tiktok.com (Login Kit + Content Posting API) | `https://<server>.ts.net/api/oauth/tiktok/callback` |
| Whop (Bounties) | whop.com/dashboard/developer | `https://<server>.ts.net/api/whop/callback` |

Logins and tokens are stored **encrypted** in `data/secrets.json`. The key is in `data/secrets.key` (owner-only; an
alternative is the environment variable `CLIPPING_SECRETS_KEY`). **Back up `data/` together with the key.** Without the key,
every connection has to be set up again.

Public TikTok posts through the official API need TikTok's app review. Until then TikTok only accepts inbox drafts
(you tap "Post" in the TikTok app).

## 5. Campaign footage onto the server

The agent looks for footage in `~/clipping/footage/` (and `~/clipping/clipping-output/_sources/`). Ways to get it there:

- **From Android:** Taildrop (Tailscale app → share file → server). On the server, `tailscale file get ~/clipping/footage/`.
- **Directly on the server:** the campaign's download link via `wget` / `curl -L -o ~/clipping/footage/<name>.mp4 "<link>"`.
- Link it to the campaign: `npm run agent -- link-source <campaignId> <file>` (or in the dashboard under the campaign).

Paths can be changed in `data/agent.json` (`sourceDirs`, `outputDir`, `openshortsOutputDir`).

## 6. Limits and notes

- **TikTok uploader (browser automation):** against TikTok's terms of use, at your own risk. It needs a one-time visible login
  window, which a server without a screen doesn't have. Not set up in this guide. Use the official TikTok route.
- **Vyro / Whop Content Rewards:** submission stays manual (no permitted interface). Whop Bounties can be automated through your own Whop app.
- **Updates:** `cd ~/clipping-agent && git pull && npm ci && npm run build && systemctl --user restart clipping-dashboard`
- **Emergency stop:** in the dashboard (Control Center) or `npm run agent -- halt`.
- **Stop everything:** `npm run agent -- uninstall-systemd`.
