# Mission Control — Flows

## 1. Start an app from the dashboard

```mermaid
sequenceDiagram
    participant U as User (dashboard)
    participant S as Control server :8088
    participant P as App process
    U->>S: POST /__hub/api/apps/vibe/start  (x-hub-token)
    S->>S: authed? · byId["vibe"]? · wired?
    S->>P: spawn (detached, own port, logs→file)
    S-->>U: { status: "starting" }
    U->>S: open SSE /apps/vibe/logs
    loop every 700ms
        S-->>U: data: <new log lines>
    end
    U->>S: GET /apps/vibe/status (poll)
    S->>P: TCP connect :5273
    P-->>S: accept ⇒ up
    S-->>U: { status: "running" }
    U->>U: boot terminal auto-closes, dot turns green
```

## 2. Live embed reaches a recruiter

```mermaid
sequenceDiagram
    participant B as Recruiter browser
    participant GH as GitHub Pages (portfolio)
    participant CF as Cloudflare Tunnel
    participant S as Control server
    participant A as Vibe frontend :5273
    B->>GH: open portfolio, click a project card
    GH-->>B: modal with <iframe src="https://<tunnel>/vibe/">
    B->>CF: GET /vibe/
    CF->>S: GET /vibe/
    S->>A: proxy /vibe/ (strip X-Frame-Options, CSP)
    A-->>S: HTML (base-pathed assets)
    S-->>B: page renders inline
    B->>CF: GET /@vite/client  (root asset, Referer /vibe/)
    CF->>S: GET /@vite/client
    S->>S: referer routing ⇒ Vibe
    S->>A: proxy (absolute)
    A-->>B: asset ⇒ app mounts live
```

## 3. Stop an app

```mermaid
sequenceDiagram
    participant U as User
    participant S as Control server
    participant P as App process-group
    U->>S: POST /__hub/api/apps/vibe/stop (token)
    alt process / script
        S->>P: process.kill(-pid)   %% whole group
    else compose
        S->>P: docker compose down
    end
    S-->>U: { status: "stopped" }
    Note over U,S: poll confirms port no longer accepts ⇒ dot grey
```

## 4. Graceful fallback when nothing is running

```mermaid
flowchart TD
    Open["Recruiter opens a project card"] --> Check{"tunnel base set<br/>& reachable?"}
    Check -->|yes| Live["Live app renders in the iframe"]
    Check -->|no| Wa["'Demos aren't live right now'<br/>▶ Request live demo → WhatsApp<br/>(pre-typed, per project)"]
    Wa --> Ping["I get the message, start the hub<br/>+ tunnel, paste URL → embeds go live"]
```

## 5. Bringing the whole portfolio up for a demo

```bash
# 1. start the control server
cd hub && HUB_TOKEN="" node server.js         # :8088

# 2. expose it publicly (no interstitial)
cloudflared tunnel --url http://localhost:8088   # → https://<name>.trycloudflare.com

# 3. from the dashboard, toggle on the apps to demo
#    (each boots on its own port; watch the SSE terminal)

# 4. paste the tunnel URL into the portfolio's "connect live demo"
#    → every project card now embeds its running app
```
