# This Portfolio — Low-Level Design

How each part is actually wired.

## 1. The site (presentation)

- **One HTML file** authored in `site/index.html`; a tiny Node wrapper prepends the `<!doctype>/<head>` skeleton to produce `docs/index.html`, which **GitHub Pages** serves.
- **Keyboard intro** — a CSS 3D keyboard; JS types the hero text, toggling a `.down` class per keystroke, then adds `.done` to lift the overlay. Plays on load; click-to-skip; disabled for `prefers-reduced-motion`.
- **Guided tour** — a spotlight `box-shadow` cutout + a positioned popover walk through hero → a project card → contact → the counters. First-visit auto-start (localStorage), replayable via a "Take a tour" button.
- **Theme** — palette as CSS custom properties; `:root[data-theme="light"]` overrides; a nav toggle flips and persists it.
- **Project modal** — built from a `projects[]` data array; renders description, a CSS **architecture schematic** from an `arch[]` spec, doc links, and (when a demo base is set) a live embed.

## 2. Engagement

```js
// unique view — increment once per browser, else just read
counted ? call("/get/"+NS+"/views")
        : call("/hit/"+NS+"/views").then(markCounted);

// reach-out — count once per browser on email / whatsapp / form-submit
a[href^="mailto:"], a[href*="wa.me"] → markReached();

// analytics
<script data-goatcounter=".../count" async src="//gc.zgo.at/count.js"></script>

// contact form → FormSubmit AJAX (no backend), spam honeypot, success toast
fetch("https://formsubmit.co/ajax/<email>", { method:"POST", body: JSON.stringify(fields) })
```

## 3. Live demos — Mission Control internals

Single zero-dependency Node server (`server.js`) on **:8088**.

### Reverse proxy (what makes embedding work)
```js
const up = http.request({ host:"127.0.0.1", port:app.port, path:p, method, headers },
  r => {
    const h = { ...r.headers };
    delete h["x-frame-options"];             // ← allow iframe embedding
    delete h["content-security-policy"];
    res.writeHead(r.statusCode, h); r.pipe(res);
  });
req.pipe(up);
```

### Referer routing (SPA assets requested at root)
A page served at `/vibe/` still asks for `/@vite/client` at root; the proxy reads the `Referer` path, matches it to an app's subpath, and routes there.

### Status, lifecycle, logs
- **status** = TCP port check (unique ports) or `docker compose ps` for compose apps.
- **start** = detached `spawn` (process group), output → per-app log file.
- **stop** = `process.kill(-pid)` (whole group) or `docker compose down`.
- **logs** = SSE tail of the log file, rendered in the dashboard's boot terminal.
- **safety** = only whitelisted commands in `apps.js`; start/stop is token-gated.

### App base-pathing (per app, so embeds render under a subpath)
```js
// vite.config: base: process.env.APP_BASE || '/'
// router:      basename = (import.meta.env.BASE_URL||'/').replace(/\/$/,'') || '/'
// start:       APP_BASE=/vibe/ npm run dev -- --port 5273 --host
```

## 4. Content & code
- **Docs** — Markdown + Mermaid per project in this public repo (`projects/<slug>/`), rendered by GitHub.
- **Code** — 9 private repos; the site links to docs, never to source.

## Embed robustness
The demo base is read from `localStorage.demoBase || DEFAULT_DEMO`, but a stored value is **ignored unless it's a valid `http(s)://` URL**, so a stale/relative value can't break the embed — the baked-in tunnel always wins.
