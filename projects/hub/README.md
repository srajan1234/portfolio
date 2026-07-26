# Mission Control — how this portfolio runs itself

This is the **meta-project**: the engineering behind the portfolio you're looking at. Every other project here isn't just described — it's a **real app you can watch running live**, embedded right inside its card. This document explains *how that works* and *why I built it this way*.

---

## The problem

I have 9 projects across wildly different stacks — Spring Boot, Play Framework, FastAPI, Node, React/Vite, multi-database. A recruiter should be able to **see them actually running**, not just read about them.

But deploying 9 full stacks to always-on cloud infrastructure is:
- **Expensive** — 9 × (backend + DB + frontend) running 24/7 for occasional views,
- **Fragile** — free tiers sleep, cold-start, and expire,
- **Pointless** — they only need to be live *while I'm demoing*.

So instead of renting cloud, I built a **control plane that runs the whole portfolio from my own machine, on demand**, and streams it to a public URL only when I want.

## The solution: one dashboard that runs everything

**Mission Control** is a single, zero-dependency Node.js server (`server.js`, ~230 lines, no npm packages) that does four jobs:

1. **Supervises processes** — starts/stops each app from a *fixed whitelist* (`apps.js`), each on its **own unique port** so all 9 can run at once.
2. **Reverse-proxies** every app under a clean subpath — `/vibe`, `/retaint`, `/cartag`, … — from one origin.
3. **Unlocks embedding** — strips `X-Frame-Options` / `CSP` headers on the way through, so each app renders inside an `<iframe>` on the public portfolio.
4. **Streams boot logs** live to the dashboard over Server-Sent Events, so you watch an app come up in a mini terminal.

An animated "Mission Control" dashboard drives it: start/stop toggles, live status dots, a boot terminal — one control panel for the entire portfolio.

## How a live demo reaches your browser

```
Recruiter's browser
   │  opens a project card on the public portfolio (GitHub Pages)
   ▼
<iframe src="https://<tunnel>/vibe/">
   │
   ▼
Cloudflare Tunnel  ──►  Mission Control (localhost:8088)
   │                        │  reverse-proxy, strip framing headers
   │                        ▼
   │                    Vibe frontend (localhost:5273, base-pathed /vibe)
   ▼
The real, running app renders inline — inside the portfolio.
```

- **Cloudflare Tunnel** gives the local hub a public HTTPS URL with no interstitial (unlike ngrok's free tier), so embeds load cleanly in an iframe.
- Each app's frontend is **base-pathed** (`APP_BASE=/vibe/` + Vite `base` + router `basename`) so its absolute asset URLs resolve correctly behind the subpath.
- The proxy uses **referer-based routing** to catch SPA assets requested at root (`/@vite/client`, `/assets/*`) and send them to the right app.

## Why it's safe

- The server **never runs arbitrary input** — only the fixed set of commands declared in `apps.js`.
- Start/stop is **token-gated** (`HUB_TOKEN`) so a tunnel visitor can view demos but can't control the machine.
- When I'm not demoing, embeds fall back to a **"request live demo" WhatsApp button** instead of a broken frame.

## Stack

| Layer | Tech |
|---|---|
| Control server | Node.js core `http` / `net` / `child_process` — **zero dependencies** |
| Process model | detached `spawn` process-groups + Docker Compose for containerized apps |
| Live logs | Server-Sent Events tailing per-app log files |
| Reverse proxy | streaming `http.request` pipe + framing-header rewrite + referer routing |
| Dashboard | vanilla HTML/CSS/JS, animated |
| Public exposure | Cloudflare Tunnel (`cloudflared`) |

## Run it

```bash
cd hub
HUB_TOKEN="" node server.js               # dashboard → http://localhost:8088
cloudflared tunnel --url http://localhost:8088   # public URL for live embeds
```

## Deeper docs

- [HLD](./HLD.md) — architecture & component responsibilities
- [LLD](./LLD.md) — server internals: status detection, proxy, SSE, base-pathing
- [FLOWS](./FLOWS.md) — start / proxy / embed / log-stream sequences
