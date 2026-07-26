# Mission Control — Low-Level Design

Everything below is in `server.js` (control server) and `apps.js` (registry). No frameworks, no dependencies.

## App registry entry (`apps.js`)

```js
{
  id: "vibe", name: "Vibe", emoji: "🧠",
  category: "Wellness · Privacy", tech: ["Spring Boot","React","FastAPI"],
  dir: `${HOME}/vibe`, port: 5273,
  subpath: "/vibe", stripPrefix: false,
  open: "http://localhost:5273/vibe/",
  start: { type: "compose", file: "docker-compose.yml", build: true },
  wired: true, boot: 60,
}
```

`start.type` is one of:
- **`process`** — `spawn(cmd, args)` directly (e.g. `node server.js`),
- **`script`** — `spawn("bash", ["-lc", cmd])` for a shell one-liner (e.g. `cd frontend && APP_BASE=/vibe/ npm run dev -- --port 5273 --host`),
- **`compose`** — `docker compose -f <file> up -d [--build]`.

## HTTP surface

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/__hub/api/apps` | list apps + live status | none |
| POST | `/__hub/api/apps/:id/start` | start an app | token |
| POST | `/__hub/api/apps/:id/stop` | stop an app | token |
| GET | `/__hub/api/apps/:id/status` | one app's status | none |
| GET | `/__hub/api/apps/:id/logs` | SSE log stream | none |
| ANY | `/{subpath}/*` | reverse proxy to app | none |
| GET | `/*` | static dashboard | none |

## Status detection

```js
async function statusOf(app) {
  if (!app.wired) return "stopped";
  if (app.start.type === "compose")
    return (await composeUp(app)) ? "running" : "stopped";
  // process/script: unique port ⇒ a port check is authoritative
  return (await portUp(app.port)) ? "running" : "stopped";
}
```

- **`portUp(port)`** opens a TCP socket to `127.0.0.1:port` with a 700 ms timeout — running if it connects.
- **`composeUp(app)`** runs `docker compose ps -q --status running` and checks for output.
- An optimistic **`starting`** state is held in memory between the start call and the port coming up.

## Process lifecycle

**Start** — detached so it survives the request, output redirected to a per-app log file:
```js
child = spawn("bash", ["-lc", s.cmd], { cwd: app.dir, detached: true,
                                        stdio: ["ignore", out, out] });
child.unref();
state[app.id] = { status: "starting", pid: child.pid };
fs.writeFileSync(pidFile, String(child.pid));   // for later kill
```

**Stop** — kill the whole process group (so child dev-servers die too):
```js
try { process.kill(-pid); }        // negative pid = process group
catch { try { process.kill(pid); } catch {} }
// compose apps: docker compose down
```

## Reverse proxy

```js
function proxy(req, res, app, absolute) {
  let p = req.url;
  if (!absolute && app.stripPrefix) {           // optional /subpath strip
    p = p.slice(app.subpath.length) || "/";
  }
  const up = http.request(
    { host:"127.0.0.1", port:app.port, path:p, method:req.method,
      headers:{ ...req.headers, host:"127.0.0.1:"+app.port } },
    r => {
      const h = { ...r.headers };
      delete h["x-frame-options"];              // ← allow iframe embedding
      delete h["content-security-policy"];
      delete h["content-security-policy-report-only"];
      res.writeHead(r.statusCode || 502, h);
      r.pipe(res);                              // stream response back
    });
  req.pipe(up);                                 // stream request body up
}
```

### Referer-based routing (the SPA problem)
A base-pathed SPA loaded at `/vibe/` still requests some assets at **root** (`/@vite/client`, `/assets/x.js`). Those don't match any subpath, so the proxy inspects the `Referer`:

```js
if (url !== "/" && !url.startsWith("/__hub/")) {
  const ref = req.headers.referer || "";
  const rp  = new URL(ref).pathname;                 // e.g. /vibe/login
  const rapp = apps.find(a => rp === a.subpath || rp.startsWith(a.subpath + "/"));
  if (rapp) return proxy(req, res, rapp, /*absolute*/ true);
}
```

This routes the stray root request to the app whose page issued it.

## Live logs over SSE

`GET /__hub/api/apps/:id/logs` opens `text/event-stream`, seeks to near the end of the app's log file, then polls every 700 ms and pushes new lines as `data:` events. The dashboard renders them in a boot terminal and auto-closes it once status flips to `running`.

## Security

- **Whitelist only** — `byId[...]` lookups; unknown ids 404. The server never interpolates network input into a command.
- **Token gate** — `authed(req)` requires `x-hub-token` on start/stop; read-only endpoints are open so the dashboard/tunnel can display state.
- **Auth toggle** — `HUB_TOKEN=""` disables the gate for purely local use; a persisted random token is generated otherwise.

## Frontend base-pathing (per app, so embeds render)

```js
// vite.config: base: process.env.APP_BASE || '/'
// router:      <BrowserRouter basename={(import.meta.env.BASE_URL||'/').replace(/\/$/,'')||'/'}>
// start cmd:   APP_BASE=/vibe/ npm run dev -- --port 5273 --host
```
This makes every asset and route resolve under the subpath the proxy serves it on.
