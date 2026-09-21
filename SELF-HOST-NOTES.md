# Self-hosted mem0 deployment notes (this fork)

Not part of upstream mem0 — deployment-specific notes for this box. Read this before touching the running stack or the patch.

## What this fork changes vs. upstream `mem0ai/mem0`

Three commits on `main`, in order:
1. `feat(server): bundle ollama as an LLM/embedder provider` — adds `ollama` to `server/requirements.txt` and to `BUNDLED_LLM_PROVIDERS`/`BUNDLED_EMBEDDER_PROVIDERS` in `server/main.py`. The underlying `mem0` SDK already implements Ollama support; upstream's server just didn't allow selecting it.
2. `feat(server): make default LLM/embedder provider selectable via env` — `DEFAULT_CONFIG`'s provider was hardcoded to `"openai"`, which crashes the container at boot (the `OpenAI()` client raises eagerly on a missing key) if you never intend to use OpenAI. Added `MEM0_LLM_PROVIDER`/`MEM0_EMBEDDER_PROVIDER`/`MEM0_EMBEDDING_MODEL_DIMS` env vars so a fully-Ollama boot works without ever touching OpenAI.
3. `fix(server): add restart policies, restrict postgres to localhost` — `mem0` and `mem0-dashboard` had no restart policy at all; Postgres's port was open to the whole LAN for no reason.
4. `fix(server): DASHBOARD_URL was hardcoded, overriding .env silently` — `mem0`'s `environment:` block hardcoded `DASHBOARD_URL=http://localhost:3020` literally, silently ignoring `.env`'s value regardless of what it said. Now interpolated from `.env`.
5. `fix(server): pass DASHBOARD_URL to the dashboard container itself` — the dashboard's own server-side `/api/auth/refresh` route reads `process.env.DASHBOARD_URL` to decide whether the session cookie should be `Secure`, but that var was never passed to the `mem0-dashboard` service at all — only to `mem0`. It silently fell back to `NODE_ENV === "production"` (true), always marking the cookie `Secure`, which browsers refuse to store over plain HTTP. This caused logins to appear to succeed (real 200, real tokens) while silently never actually persisting a session — an immediate bounce back to `/login?next=...` on the next navigation. Root-caused by hand-simulating the full browser login flow (login → the dashboard's own cookie-set call) and inspecting the raw `Set-Cookie` header.
6. `feat(server): point dashboard/CORS at the public bbmind.brbjr.com domain` — see "Public exposure" below.
7. `fix(server): move public API path off /api to avoid dashboard collision` — the dashboard's own internal Next.js route `/api/auth/refresh` (same app, sets the session cookie) got hijacked once NPM started forwarding everything under `/api/` to the external mem0 API server instead. Moved the public API path to `/mem0-api` to stop colliding with Next.js's reserved `/api/*` namespace.

## Current deployment

- Directory: `/mnt/user/appdata/mem0` (this repo, root = compose build context's parent via `server/docker-compose.yaml`'s `context: ..`)
- Run from `server/`: `docker compose up -d --build`
- API: `http://localhost:8140` (remapped from upstream's default 8888 — that port was taken by another container on this host); also reachable publicly as `https://bbmind.brbjr.com/mem0-api/*` (see "Public exposure" below — NOT `/api/*`, that collides with the dashboard's own routes).
- Dashboard: `http://localhost:3020` (remapped from 3000, same reason); the primary access path is now `https://bbmind.brbjr.com` — see below, direct LAN-IP access to the dashboard no longer fully works for login.
- Postgres: `127.0.0.1:8432` only (not LAN-reachable)
- **LLM/embedder: fully local Ollama**, not OpenAI. `qwen3:4b` was tried first for extraction and was too weak to reliably follow mem0's structured-extraction prompt (returned empty results on clear, extractable facts); `qwen3:8b` works correctly. `nomic-embed-text` for embeddings (768-dim, matches `MEM0_EMBEDDING_MODEL_DIMS=768` in `.env`).
- Ollama reached via `host.docker.internal` (the `ollama` container is on the default bridge network, not shared with this compose stack, and doesn't support container-name DNS resolution anyway) — same fix `bbmind` needed.
- `.env` (gitignored, not in this repo) holds: `JWT_SECRET`, `POSTGRES_PASSWORD`, the Ollama provider/model/dims vars, `DASHBOARD_URL=http://localhost:3020`.
- Admin account: originally seeded as `brbjr1@gmail.com` via `server/scripts/seed.sh`; email and password were changed by the user directly in the dashboard afterward (current login not recorded here — it's a live credential, use the dashboard's own password reset if lost: `make reset-admin-password EMAIL=<email> PASSWORD=<new>` per `server/Makefile`). First API key (`m0sk_...`) from the original seed is what `mem0-mcp-bridge` uses (see below) and is unaffected by the account email/password change. Not recorded here either; check `~/.claude.json`'s `mcpServers.mem0-bridge.env.MEM0_API_KEY` or generate a new one from the dashboard if lost.

## Scope decision (this session)

- **Single-user**, not multi-user/family. The original ask was a family "second brain," but real multi-user security here would require per-person API keys/accounts that are actually *restricted* to their own `user_id` server-side — verified this isn't automatically enforced (any caller holding an API key can pass any `user_id` in a request) and that work was never done. If family use comes back later, that restriction needs building/verifying before treating per-user scoping as real security, not just data organization.
- **Public exposure: live and confirmed working end-to-end** (real browser login, not just curl), reusing the existing `bbmind.brbjr.com` NPM proxy host (ID 71, previously pointed at the now-superseded `bbmind` app) and its existing Let's Encrypt certificate (ID 81) — no new cert needed. Root `/` → dashboard (`192.168.20.15:3020`); `/mem0-api/` → API (`192.168.20.15:8140`), prefix stripped via a `rewrite ^/mem0-api/(.*)$ /$1 break;` directive in the Custom Location's Advanced box. `NEXT_PUBLIC_API_URL`/`DASHBOARD_URL` point at this domain now, so dashboard and API are effectively same-origin from the browser's perspective (no more CORS in practice, since both live under one domain). **Trade-off:** direct LAN-IP dashboard access (`http://192.168.20.15:3020`) no longer fully works for login — the bundled JS calls the public HTTPS API URL specifically. Access is via the public domain from any device except this Docker host itself (hairpin NAT isn't supported by the router here, confirmed identical to the sibling `bbhealth` deployment's known issue — testing must happen from another LAN device or the internet, not from a shell on this box).

  **Two NPM-side gotchas hit while setting this up (not code, but worth knowing if this ever needs redoing):**
  - NPM's Custom Location template (`/opt/nginx-proxy-manager/templates/_location.conf` inside the `NginxProxyManager` container) *always* emits its own `proxy_pass {{forward_scheme}}://{{forward_host}}:{{forward_port}}` line from the Forward Hostname/Port fields, in addition to whatever raw text goes in the Advanced box. Putting a `proxy_pass` line in Advanced too creates a duplicate directive — a fatal `nginx -t` error (`"proxy_pass" directive is duplicate`) that aborts the *entire* reload and silently deletes that host's generated `.conf` file (found via `docker exec NginxProxyManager tail /config/log/error.log`, since the file itself just vanishes with no UI-visible error). Use a `rewrite ... break;` directive in Advanced instead of `proxy_pass` for prefix-stripping — it runs before the template's own auto-generated `proxy_pass`, achieving the same result without the conflict.
  - Don't reuse the path `/api` for anything proxied via NPM to an app whose own frontend (Next.js here) reserves `/api/*` for its own server routes — the NPM location will intercept and misroute the app's own internal API calls. Learned this the hard way (`/mem0-api` was the fix, not `/api`).

## Claude Code integration

- `/mnt/user/appdata/mem0-mcp-bridge/mem0_bridge.py` — a **pure standard-library** MCP stdio bridge (no `mcp`/`httpx` packages; this host's system Python 3.9 has no SSL support at all, so `pip install` of anything from PyPI fails here — confirmed even `import ssl` fails). Implements the MCP JSON-RPC protocol by hand: `initialize`, `tools/list`, `tools/call` for `add_memory`/`search_memory`/`list_memories`/`delete_memory`, calling mem0's plain-HTTP REST API (`/memories`, `/search`) directly.
- Registered globally in `~/.claude.json`'s top-level `mcpServers.mem0-bridge` (not a per-project `.mcp.json`) — applies to every Claude Code session on this box. **Requires restarting Claude Code to take effect** in any already-running session.
- `~/.claude/CLAUDE.md` (global, applies everywhere) tells every session to proactively search/contribute personal facts, not just on request.
- The official `mem0ai/mem0` Claude Code plugin (`integrations/claude-code-plugin/`) was evaluated and **rejected** — it only talks to mem0's hosted cloud platform (`api.mem0.ai`, `/v1/`/`/v2/`/`/v3/` endpoints), not this self-hosted server's simpler REST API (`/memories`, `/search`). Not a fit without real engineering work on either side.

## Relationship to `bbmind`

`bbmind` (`/mnt/user/appdata/bbmind`) was the first attempt at this — a hand-rolled FastAPI/SQLite facts+notes app. This mem0 deployment is evaluating whether to replace it. As of this session: mem0 spike succeeded and this became the real deployment, but `bbmind` has **not** been decommissioned — no data has been migrated, and it's still running. Revisit once mem0 has some real-world track record.

## Known gaps / next steps

- MCP bridge confirmed connected after a Claude Code restart (`mcp__mem0-bridge__*` tools visible); not yet exercised end-to-end in a real coding session, only the scripted stdio test done during setup.
- Public login confirmed working end-to-end via the user's own real browser (not just curl) — but not yet tested from an actual external (off-LAN/off-WiFi) device, only whatever network the user tested from.
- No backup strategy for the Postgres volume (`mem0-dev_postgres_db`) — worth adding to whatever backup routine covers the rest of this box's appdata.
- `bbmind`'s fate undecided — its own NPM entry (`bbmind.brbjr.com`, ID 71) was repointed to this mem0 deployment in this session, so `bbmind` the app no longer has a public URL of its own. It's still running on the LAN at `192.168.20.15:8130` but effectively superseded.
