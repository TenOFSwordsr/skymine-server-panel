# SkyMine Panel - Docker Minecraft Admin Panel

A small self-hosted control panel for Minecraft servers running as Docker containers, plus the
Velocity plugin that adds network teleport shortcuts. The backend is a single Go binary with no
framework and no database: it talks to the Docker socket directly and serves an embedded Svelte
SPA alongside its JSON API. Built to run one specific server (container `skymine-mc-1`), so the
server-to-path mapping is hardcoded rather than configured.

**Suggested repo name:** `skymine-server-panel`
**Stack:** Go 1.26 (`net/http` stdlib only, `go:embed`), Svelte 5 + Vite, Velocity proxy API (Java) with Adventure
**Status:** active
**Last modified:** 2026-09-15

## What it does

- `backend/` - HTTP panel on `:3000` (override with `PANEL_ADDR`). Uses the Docker Engine REST API
  v1.44 over `/var/run/docker.sock` (path overridable with `DOCKER_SOCK`) for:
  - `GET /api/health`, `GET /api/servers` - container name/state/status/image list
  - `POST /api/servers/{name}/power` - `start` / `stop` / `restart` / `kill`
  - `GET /api/servers/{name}/logs` - tails Docker logs and strips the 8-byte multiplexed frame headers
  - `GET|POST|DELETE /api/servers/{name}/plugins` - lists, uploads and deletes `.jar` files
  - `GET /api/servers/{name}/files`, `GET|PUT /api/servers/{name}/file`, `/download`,
    `/file-upload`, `/mkdir`, `/rename`, `DELETE /path` - a full file manager over the server's
    `mc-data` tree, with path-traversal rejection in `safePath`
  - `POST /api/servers/{name}/command` - returns 501, console input is not implemented
  - the built frontend is embedded from `backend/dist` and served with an SPA fallback
- `frontend/` - one-page Svelte 5 app: server list with auto-refresh, live log tail, plugin
  upload/delete, power buttons.
- `velocity-plugin/SkyMineCommandsPlugin.java` - Velocity plugin registering `/lobby` and
  `/skymine` as player-only server shortcuts via `connectWithIndication()`.

## Layout

```
backend/main.go            routes, embedded SPA, JSON helpers
backend/docker.go          Docker socket client, power/logs/plugins handlers
backend/files.go           data root mapping, safePath, file read/write
backend/files_handlers.go  file manager endpoints
backend/util.go            small shared helpers
backend/dist/              built SPA, embedded at compile time
backend/skymine-panel      compiled Linux binary (build output)
frontend/src/App.svelte    the whole UI
velocity-plugin/           SkyMineCommandsPlugin.java
```

## Running it

```bash
cd frontend && npm install && npm run build     # vite.config.js outputs to ../backend/dist
cd ../backend && go run .                       # PANEL_ADDR=:8080 to move the port
```

## Notes

- There is no authentication or authorisation anywhere in the API, and the preflight handler sets
  `Access-Control-Allow-Origin: *`. It is meant for same-origin use behind a trusted network -
  do not expose it.
- `pluginDir()` and `dataRoot()` only know about `skymine-mc-1` / `skymine` and point at
  `/app/servers/skymine/mc-data`; add your own cases for any other container.
- `backend/skymine-panel` (6.9 MB) and `backend/dist/` are build artifacts and should be ignored.
- `velocity-plugin/` has no build file - it compiles against the Velocity API jar and
  `jakarta.inject`, so it needs a Gradle/Maven setup before it can be built.
