# This Portfolio — how it's built

This document is about the site you're looking at *itself*, as an engineering project: how it's developed, what it's made of, and all its parts. It's meta — the portfolio is one of the projects in the portfolio.

The goal wasn't "a page with my projects on it." It was: **let a recruiter see my projects actually running, feel the craft, and reach me — all from one link, with zero always-on cloud cost.**

---

## The parts

The system has four layers. Each is deliberately simple and dependency-light.

### 1. Presentation — the animated site
- A **single hand-authored HTML file** (no framework, no build step) served as a static site on **GitHub Pages**.
- A **live-typed keyboard intro**: a 3D mechanical keyboard types "Backend Engineer" on load, keys depressing, then lifts away to reveal the hero.
- A **guided tour** (spotlight + coach-marks) for first-time visitors, replayable anytime.
- **Light / dark theming** via `data-theme`, persisted per visitor.
- Project cards open a **modal** with description, a **live architecture schematic**, and design-doc links.

### 2. Engagement — knowing it's working
- **Live counters** in the nav — unique views and reach-outs — backed by a privacy-friendly hit counter.
- **GoatCounter** analytics (no cookies, GDPR-clean) for referrers, geography, and device.
- A **contact pipeline**: an inline form (no backend, via FormSubmit) plus a **WhatsApp** button with a pre-typed message.

### 3. Live demos — Mission Control (the hub)
This is the part that makes every *other* project a running app inside its card:
- A **zero-dependency Node control plane** starts/stops and **reverse-proxies** all 9 apps on demand, each on its own port.
- It **strips framing headers** so each app renders in an `<iframe>` on this site, and **streams boot logs** over SSE to an animated dashboard.
- A **Cloudflare Tunnel** publishes it at a public HTTPS URL (no interstitial), so demos stream live into the portfolio.
- When nothing's running, embeds fall back to a **"request live demo" WhatsApp** button instead of a dead frame.

### 4. Content & code
- **Design docs** (HLD / LLD / FLOWS per project, with Mermaid diagrams) live in this **public docs repo** — so the *ideas* are shown without exposing source.
- The actual **source code lives in private repos** — visible on request, not scraped from a public site.

## What's used

| Concern | Choice |
|---|---|
| Site | Hand-written HTML/CSS/JS, GitHub Pages |
| Diagrams | Mermaid (rendered on GitHub) + inline CSS schematics |
| Live demos | Node.js (zero-dep) control plane + reverse proxy |
| Public exposure | Cloudflare Tunnel (`cloudflared`) |
| App embedding | Vite `base` + router `basename` per app (base-pathing) |
| Analytics | GoatCounter + a lightweight hit counter |
| Contact | FormSubmit (form) + WhatsApp deep link |
| Design/animation | custom, calibrated per section |

## Design principles

- **Zero always-on cost** — demos run from my machine only while I'm showing them.
- **Dependency-light** — the site and the control plane both avoid frameworks so they're portable and auditable.
- **Ideas public, code private** — docs and live behavior are open; source is shared deliberately.
- **Graceful when off** — nothing looks broken when the demos aren't live.

## Deeper docs

- [HLD](./HLD.md) — the four layers and how a live demo reaches a browser
- [LLD](./LLD.md) — site internals, the control plane, base-pathing, analytics wiring
- [FLOWS](./FLOWS.md) — load / demo / contact / fallback sequences
