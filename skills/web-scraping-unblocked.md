---
name: web-scraping-unblocked
description: "Fetch pages that block bots: strategy tiers, Camoufox anti-detect service, toolset routing."
tags: [web-scraping, anti-detect, proxy, camoufox, residential-ip, hermes]
---

# Web Scraping: Getting Past Bot Detection

Use when a page returns 403, Cloudflare challenge, CAPTCHA, or empty content.

## Triage: What's Actually Blocking You?

Before reaching for heavy tools, identify the block type:

1. **Datacenter IP block** — request comes from a cloud/VPS subnet (most common for Hermes running in Docker on a homelab: this is usually NOT the problem since the homelab is on a residential IP)
2. **Browser fingerprint detection** — headless Chrome/Firefox has distinct TLS/JA3, canvas, WebGL, and CDP fingerprints
3. **Cloudflare JS challenge** — requires JavaScript execution and cookie setting
4. **Behavioral detection** — timing, mouse movement, scroll patterns
5. **Rate limiting** — too many requests too fast

## Routing Strategy (try in order)

### Tier 1: `web` toolset via delegate_task (free, try first)
```python
delegate_task(
    goal="Fetch content from <url> using web_search and web_extract",
    toolsets=["web"]
)
```
`web_extract` uses a lighter HTTP stack than the browser. Gets through many sites that block the Chromium browser stack. **Try this before anything else.**

### Tier 2: Camoufox service (self-hosted on homelab)
For sites that require full browser fingerprinting. Service is at `/opt/data/camoufox-service/`.

```bash
# Fetch rendered HTML
curl "http://camoufox:3001/fetch?url=https://example.com"

# Screenshot
curl "http://camoufox:3001/screenshot?url=https://example.com" --output shot.png
```

From inside Docker network: `http://camoufox:3001`
From host: `http://localhost:3001`

See `references/camoufox-service.md` for full architecture, deployment notes, and SSH deployment from Hermes.
See `references/homelab-ssh.md` for SSH access patterns, file copying, and docker commands on containership.

### Tier 3: Consider IPRoyal or similar
Only if the homelab residential IP itself is blocked (rare). ~$2-3/GB, no minimums. Oxylabs is enterprise-grade but overkill for casual use.

## Key Insight: Why Residential IP Often Isn't Enough

Modern anti-bot systems check the **full fingerprint stack**, not just IP:
- **JA3/JA4 TLS fingerprint** — headless Chrome/Firefox has distinct signatures
- **User-Agent vs. browser bugs** — detection systems verify known browser bugs match the claimed UA
- **Canvas/WebGL rendering** — GPU-rendered values differ between headless and real browsers
- **HTTP/2 fingerprint** — frame ordering, header priority differs by browser
- **Behavioral signals** — mouse movement, timing, scroll patterns

Camoufox patches all of these at the Firefox level. It's not just spoofing headers — it's actually making Firefox behave differently at the rendering and protocol layer.

## Camoufox vs. Alternatives

| Tool | IP | Fingerprint | JS | Self-hosted | Cost |
|---|---|---|---|---|---|
| `web_extract` | datacenter | none | no | yes | free |
| Browserless | datacenter | partial | yes | yes | free |
| Camoufox (self-hosted) | residential (homelab) | full Firefox | yes | yes | free |
| Oxylabs residential | residential pool | none (just proxy) | no | no | $8/GB |
| Oxylabs Scraper API | residential pool | partial | yes | no | $2/1k results |

## Notes
- No official Camoufox Docker image exists — the service at `/opt/data/camoufox-service/` is custom-built
- The `camoufox fetch` step (~100MB Firefox binary) runs at `docker build` time, needs internet access
- Reddit: Camoufox gets through IP/fingerprint detection but still hits a JS verification challenge — needs persistent browser profile to cache solved session cookies. See `references/camoufox-service.md`.
- `headless='virtual'` uses Xvfb internally; no manual display server setup needed
- **Hermes terminal broken:** `MESSAGING_CWD=/home/node/.openclaw/workspace` env var in Hermes container (OpenClaw artifact). Fix: change to `MESSAGING_CWD=/opt/data` in compose. Workaround until fixed: pass `workdir="/opt/data"` in every delegate_task+terminal subagent.
