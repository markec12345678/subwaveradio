# SUB/WAVE — Quick Start Guide

**Goal:** Get a working SUB/WAVE radio station on-air in under 30 minutes, end to end.

This guide covers the recommended path: Docker Compose on a single Linux host, Cloudflare for TLS, Navidrome for the music library, and a cloud LLM provider (OpenAI/Anthropic) or local Ollama for the AI DJ.

---

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Linux x86_64 or arm64 | Debian 12 / Ubuntu 24.04 |
| RAM | 2 GB | 4 GB (8 GB if running local Ollama) |
| Disk | 10 GB free | 50 GB+ (music library + archives) |
| Docker | 24+ | Latest |
| Docker Compose | v2+ | Latest |
| Navidrome | 0.55+ | 0.62+ (security fixes + `sonicSimilarity`) |
| LLM | Any cloud provider API key OR local Ollama | OpenAI gpt-4o-mini or Anthropic Claude Haiku (cheap, fast) |
| Domain | Optional but recommended | Cloudflare-managed for TLS + DDoS |

---

## Step 1: Install Navidrome (5 min)

Navidrome is your music library server. SUB/WAVE connects to it over the Subsonic API.

```bash
mkdir -p ~/navidrome/music ~/navidrome/data
cd ~/navidrome

cat > docker-compose.yml << 'EOF'
services:
  navidrome:
    image: deluan/navidrome:latest
    ports:
      - "4533:4533"
    volumes:
      - ./data:/data
      - ./music:/music:ro
    environment:
      - ND_LOGLEVEL=info
      - ND_SESSIONTIMEOUT=24h
      - ND_BASEURL=""
    restart: unless-stopped
EOF

docker compose up -d
```

Copy your music files into `~/navidrome/music/`, then open `http://<your-host>:4533` and create an admin account.

**Note the credentials** — you'll need them in Step 4.

---

## Step 2: Install SUB/WAVE via CLI (2 min)

The CLI is the recommended path — single binary, no Node.js on the host, no git clone required.

```bash
curl -fsSL https://cli.getsubwave.com | sh
```

The installer prompts:
- `Run subwave init now?` → **Yes**
- `Bring the stack up now?` → **Yes**

`subwave init` scaffolds `~/subwave/` with:
- `docker-compose.yml` (bundled Caddy variant)
- `.env` (3 vars pre-filled: `ADMIN_USER=admin`, random `ADMIN_PASS`, empty `SITE_URL`)
- `state/` directory (bind-mounted into all containers)

Edit `.env`:

```bash
cd ~/subwave
nano .env
```

```dotenv
ADMIN_USER=admin
ADMIN_PASS=<the random value, or your own>
SITE_URL=https://radio.yourdomain.com   # set later if no domain yet
```

---

## Step 3: Start the Stack (2 min)

```bash
subwave start
# or: docker compose up -d
```

Verify services are healthy:

```bash
subwave status
```

Expected output:
```
✓ caddy        healthy   :7700
✓ broadcast    healthy   icecast + liquidsoap
✓ controller   healthy   :7701
✓ web          started   Next.js
✓ analyzer     healthy   librosa
✓ docker-proxy started   read-only
```

If `controller` shows `unhealthy`, check logs:

```bash
subwave logs controller
```

Common issues:
- `ADMIN_USER/ADMIN_PASS missing` → set them in `.env`, then `subwave restart controller`
- `Port 7700 already in use` → change `CADDY_PORT` in `.env`

---

## Step 4: Run the Onboarding Wizard (10 min)

Open the browser:

```bash
subwave admin
# opens https://<your-host>/onboarding (or http:// if no SITE_URL yet)
```

Log in with `ADMIN_USER` / `ADMIN_PASS` from `.env`. The wizard collects:

### 4.1 Navidrome connection
- **URL:** `http://host.docker.internal:4533` (if Navidrome is on the same host)
- **Username:** your Navidrome admin
- **Password:** your Navidrome admin password

The wizard tests the connection and scans your library.

### 4.2 LLM provider

Pick one:

| Provider | Cost | Latency | Quality | Setup |
|---|---|---|---|---|
| **Ollama** (local) | Free | 2-10s/pick | Medium | Install Ollama on host, pull `qwen2.5:7b` or `llama3.1:8b` |
| **OpenAI** | ~$0.01/day | 1-3s/pick | High | Paste `OPENAI_API_KEY` |
| **Anthropic** | ~$0.01/day | 1-3s/pick | Highest | Paste `ANTHROPIC_API_KEY` |
| **Google Gemini** | Free tier | 1-2s/pick | High | Paste `GOOGLE_GENERATIVE_AI_API_KEY` |
| **DeepSeek** | Cheapest cloud | 2-5s/pick | Medium | Paste `DEEPSEEK_API_KEY` |
| **OpenRouter** | Aggregator | varies | varies | Paste `OPENROUTER_API_KEY`, pick any model |

**Recommendation for first-timers:** OpenAI `gpt-4o-mini` — cheap (~$0.01-0.05/day for a personal station), fast, high quality. Anthropic `claude-haiku-3.5` is equivalent.

**For local/offline:** Ollama with `qwen2.5:7b-instruct` — needs 8 GB RAM, runs on CPU acceptably.

### 4.3 TTS engine

| Engine | Quality | Setup |
|---|---|---|
| **Piper** (default) | Basic, robotic | None — bundled in controller image |
| **Kokoro** | Natural | None — bundled in controller image |
| **OpenAI TTS** | High | Needs OpenAI key (same as LLM) |
| **ElevenLabs** | Highest | Needs `ELEVENLABS_API_KEY` |
| **Chatterbox** | Voice cloning | Needs `--profile tts-heavy` (PyTorch, ~5 GB) |
| **PocketTTS** | Multilingual | Needs `--profile tts-heavy` + optional `HF_TOKEN` |

**Recommendation:** Start with Piper (default). Upgrade to Kokoro or OpenAI TTS after you've confirmed the stack works.

### 4.4 DJ persona

Create your first DJ:
- **Name:** e.g. "Echo"
- **Voice:** pick from Piper voices (alan, alba, etc.)
- **Personality:** 2-3 sentences describing the DJ's style
- **System prompt:** optional — leave blank for default

You can add up to 10 personas later from `/admin/personas`.

### 4.5 Jingles (optional)

The wizard offers to generate station ID jingles. Skip if you don't have OpenAI/ElevenLabs — defaults are seeded on first boot.

---

## Step 5: Tune In (1 min)

### Web player
```bash
subwave listen
# opens https://<your-host>/listen
```

### Sonos / VLC / car radio
Paste this URL into your player:
```
https://<your-host>/listen.pls
```
or
```
https://<your-host>/listen.m3u
```

### Mobile apps
- **iOS:** [App Store](https://apps.apple.com/app/sub-wave/id6778786696)
- **Android:** [Google Play](https://play.google.com/store/apps/details?id=com.getsubwave.app)

Add your station URL when prompted.

---

## Step 6: Optional — Cloudflare TLS (5 min)

For a public station with HTTPS:

1. **Buy/transfer a domain** to Cloudflare (or just change nameservers)
2. **Add an A record** pointing at your host's public IP, proxied (orange cloud)
3. **Set `SITE_URL`** in `~/subwave/.env`:
   ```dotenv
   SITE_URL=https://radio.yourdomain.com
   ```
4. **Restart Caddy:**
   ```bash
   subwave restart caddy
   ```
5. Cloudflare's edge cert covers TLS. Caddy listens on HTTP internally.

### Optional — Cloudflare Access (private station)
For a station only you (and invited people) can access:
1. Cloudflare dashboard → Zero Trust → Access → Applications → Add application
2. Self-hosted, domain `radio.yourdomain.com`
3. Policy: allow your email
4. IdP: Cloudflare's built-in, or Google/GitHub
5. Now `/listen`, `/stream.mp3`, `/admin` all require IdP login

This is the recommended way to run a private station without music-licensing concerns.

---

## Step 7: Optional — 24×7 Schedule (5 min)

A schedule lets different DJ personas take over at different times.

1. Open `/admin/shows`
2. Click a slot in the 24×7 grid
3. Pick a persona, mood (low/medium/high energy), optional genre filter
4. Save

Example: a "Morning Coffee" show 6-9am with a calm persona and low energy; a "Friday Night" show 22-1am with a high-energy persona and electronic genre.

---

## Step 8: Optional — Custom Skills (10 min)

Skills are the DJ's between-track segments. 7 built-ins ship by default (weather, news, curiosity, web-search, now-playing-dig, album-anniversary, library-deep-cut). To add your own:

```bash
mkdir -p ~/subwave/state/skills/on-this-day
cd ~/subwave/state/skills/on-this-day
```

Create `SKILL.md`:
```markdown
---
name: on-this-day
label: On This Day
cooldown: 45m
context: date, clock
---
A brief "on this day in music history" fact. Pick one event from the date,
mention the year and the artist, frame it as a DJ would — casual, not
Wikipedia. Never repeat a fact within 30 days.
```

Optional `tool.mjs` for data fetching:
```javascript
export default async (ctx, state, services, config, input) => {
  // services.search, services.library, services.recall available
  const today = new Date().toISOString().slice(5, 10); // MM-DD
  const r = await fetch(`https://api.wikimedia.org/feed/1.0/onthisday/events/${today}`);
  const data = await r.json();
  return { events: data.events?.slice(0, 5) ?? [] };
};
```

Then in `/admin/skills` → Rescan → toggle on. See `docs/custom-skills.md` for the full contract.

---

## Troubleshooting

### Controller won't start
```bash
subwave logs controller | head -50
```
- `ADMIN_USER/ADMIN_PASS missing` → set in `.env`, `subwave restart controller`
- `Navidrome unreachable` → check URL in `/admin/settings`, ensure `host.docker.internal` resolves (`extra_hosts` is in compose)
- `Ollama connection refused` → ensure Ollama is running on the host, `OLLAMA_HOST=0.0.0.0:11434`

### No audio on stream
```bash
subwave logs broadcast | tail -30
```
- Liquidsoap "no source" → queue is empty, wait 30s for first pick
- Icecast "source not connected" → check `state/icecast-secrets.env` exists; if not, `subwave restart broadcast`

### DJ never talks
- Check `/admin/dash` → booth log for DJ events
- Verify LLM provider in `/admin/settings` → test connection
- Check `state/events.json` for LLM errors
- If using Ollama, ensure the model is pulled: `ollama pull qwen2.5:7b`

### Library not scanning
- `/admin/library` → Rescan
- Check Navidrome has music in `/music` and Navidrome itself scanned it
- Look for Subsonic API errors in controller logs

### Memory issues
```bash
docker stats
```
- Analyzer > 6 GB → it's analyzing a large library; wait or `docker compose stop analyzer`
- TTS-heavy > 10 GB → reduce concurrent TTS, or add GPU

### Stream buffering / dropouts
- Check upload bandwidth (128 kbps MP3 + 96 kbps Opus = ~224 kbps per listener)
- Cloudflare caches static assets but not the stream — for > 50 listeners, consider an Icecast relay
- Liquidsoap 2.4.5 is pinned for a reason — don't downgrade

---

## Next Steps

- **Library Observatory** — visit `/observatory` for a data-art map of your tagged library
- **MCP server** — connect Claude Desktop / Cursor to drive the DJ; see `mcp-subwave/README.md`
- **Community station** — submit yours at `getsubwave.com/stations` (open a GitHub issue with the `add-station` template)
- **Backup** — `/admin/backup` downloads a ZIP of the entire state; encrypt before off-site storage
- **Update** — `subwave update` pulls new images and recreates changed services

---

## Uninstall

```bash
subwave uninstall           # stops stack, removes containers
subwave uninstall --purge   # also removes state/ and volumes
subwave uninstall --images  # also removes pulled GHCR images
subwave uninstall --binary  # also removes the subwave binary
```

---

*For full architecture and security details, see `README.md`, `CLAUDE.md`, and `SECURITY_AUDIT.md`.*
