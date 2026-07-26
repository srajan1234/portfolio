# This Portfolio — Flows

## 1. First page load

```mermaid
sequenceDiagram
    participant B as Browser
    participant GH as GitHub Pages
    participant GC as Analytics
    B->>GH: GET /portfolio/
    GH-->>B: static HTML (one file)
    B->>B: keyboard intro types "Backend Engineer" → lifts
    B->>B: first visit? → guided tour (spotlight coach-marks)
    B->>GC: register unique view (once per browser)
    B->>B: apply saved light/dark theme
```

## 2. Exploring a project + live demo

```mermaid
sequenceDiagram
    participant B as Browser
    participant CF as Cloudflare Tunnel
    participant HUB as Mission Control
    participant A as App (e.g. Vibe)
    B->>B: click card → modal (desc, schematic, doc links flash)
    alt tunnel connected
        B->>CF: iframe GET /vibe/
        CF->>HUB: /vibe/
        HUB->>A: proxy (strip framing headers)
        A-->>B: real app renders inline
    else not connected
        B->>B: "Demos aren't live" → ▶ Request live demo (WhatsApp, pre-typed)
    end
```

## 3. Reaching out

```mermaid
flowchart TD
    V["Visitor wants to connect"] --> Pick{"How?"}
    Pick -->|Form| F["Inline form → FormSubmit → my inbox"]
    Pick -->|WhatsApp| W["wa.me deep link, pre-typed message"]
    Pick -->|Email / socials| E["mailto / LinkedIn / GitHub"]
    F --> C["reach-out counter +1 (once per browser)"]
    W --> C
    E --> C
```

## 4. Bringing the whole thing live for a demo

```bash
# 1. control plane
cd hub && HUB_TOKEN="" node server.js           # :8088

# 2. public URL (no interstitial)
cloudflared tunnel --url http://localhost:8088   # → https://<name>.trycloudflare.com

# 3. toggle apps on from the dashboard (each on its own port, SSE boot terminal)

# 4. paste the tunnel URL into the portfolio → every card embeds its running app
```

## 5. What a visitor sees when demos are off
- Site, intro, tour, docs, counters, analytics, contact — **all still fully work** (they're public/always-on).
- Only the in-card live previews swap to a **"request live demo" WhatsApp** button — nothing looks broken.
