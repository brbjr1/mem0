# Self-hosted mem0 deployment notes (this fork)

Not part of upstream mem0 — deployment-specific notes for this box. Read this before touching the running stack or the patch.

## What this fork changes vs. upstream `mem0ai/mem0`

Three commits on `main`, in order:
1. `feat(server): bundle ollama as an LLM/embedder provider` — adds `ollama` to `server/requirements.txt` and to `BUNDLED_LLM_PROVIDERS`/`BUNDLED_EMBEDDER_PROVIDERS` in `server/main.py`. The underlying `mem0` SDK already implements Ollama support; upstream's server just didn't allow selecting it.
2. `feat(server): make default LLM/embedder provider selectable via env` — `DEFAULT_CONFIG`'s provider was hardcoded to `"openai"`, which crashes the container at boot (the `OpenAI()` client raises eagerly on a missing key) if you never intend to use OpenAI. Added `MEM0_LLM_PROVIDER`/`MEM0_EMBEDDER_PROVIDER`/`MEM0_EMBEDDING_MODEL_DIMS` env vars so a fully-Ollama boot works without ever touching OpenAI.
3. `fix(server): add restart policies, restrict postgres to localhost` — `mem0` and `mem0-dashboard` had no restart policy at all; Postgres's port was open to the whole LAN for no reason.

## Current deployment

- Directory: `/mnt/user/appdata/mem0` (this repo, root = compose build context's parent via `server/docker-compose.yaml`'s `context: ..`)
- Run from `server/`: `docker compose up -d --build`
- API: `http://localhost:8140` (remapped from upstream's default 8888 — that port was taken by another container on this host)
- Dashboard: `http://localhost:3020` (remapped from 3000, same reason)
- Postgres: `127.0.0.1:8432` only (not LAN-reachable)
- **LLM/embedder: fully local Ollama**, not OpenAI. `qwen3:4b` was tried first for extraction and was too weak to reliably follow mem0's structured-extraction prompt (returned empty results on clear, extractable facts); `qwen3:8b` works correctly. `nomic-embed-text` for embeddings (768-dim, matches `MEM0_EMBEDDING_MODEL_DIMS=768` in `.env`).
- Ollama reached via `host.docker.internal` (the `ollama` container is on the default bridge network, not shared with this compose stack, and doesn't support container-name DNS resolution anyway) — same fix `bbmind` needed.
- `.env` (gitignored, not in this repo) holds: `JWT_SECRET`, `POSTGRES_PASSWORD`, the Ollama provider/model/dims vars, `DASHBOARD_URL=http://localhost:3020`.
- Admin account: `brbjr1@gmail.com`, seeded via `server/scripts/seed.sh`. First API key (`m0sk_...`) is also in that seed output — same key is what `mem0-mcp-bridge` uses (see below). Not recorded here since it's a live credential; check `~/.claude.json`'s `mcpServers.mem0-bridge.env.MEM0_API_KEY` or generate a new one from the dashboard if lost.

## Scope decision (this session)

- **Single-user**, not multi-user/family. The original ask was a family "second brain," but real multi-user security here would require per-person API keys/accounts that are actually *restricted* to their own `user_id` server-side — verified this isn't automatically enforced (any caller holding an API key can pass any `user_id` in a request) and that work was never done. If family use comes back later, that restriction needs building/verifying before treating per-user scoping as real security, not just data organization.
- **Public exposure: none yet.** Explicitly deferred putting this behind NPM+SSL — LAN-only for now, revisit after the MCP workflow proves out over real use.

## Claude Code integration

- `/mnt/user/appdata/mem0-mcp-bridge/mem0_bridge.py` — a **pure standard-library** MCP stdio bridge (no `mcp`/`httpx` packages; this host's system Python 3.9 has no SSL support at all, so `pip install` of anything from PyPI fails here — confirmed even `import ssl` fails). Implements the MCP JSON-RPC protocol by hand: `initialize`, `tools/list`, `tools/call` for `add_memory`/`search_memory`/`list_memories`/`delete_memory`, calling mem0's plain-HTTP REST API (`/memories`, `/search`) directly.
- Registered globally in `~/.claude.json`'s top-level `mcpServers.mem0-bridge` (not a per-project `.mcp.json`) — applies to every Claude Code session on this box. **Requires restarting Claude Code to take effect** in any already-running session.
- `~/.claude/CLAUDE.md` (global, applies everywhere) tells every session to proactively search/contribute personal facts, not just on request.
- The official `mem0ai/mem0` Claude Code plugin (`integrations/claude-code-plugin/`) was evaluated and **rejected** — it only talks to mem0's hosted cloud platform (`api.mem0.ai`, `/v1/`/`/v2/`/`/v3/` endpoints), not this self-hosted server's simpler REST API (`/memories`, `/search`). Not a fit without real engineering work on either side.

## Relationship to `bbmind`

`bbmind` (`/mnt/user/appdata/bbmind`) was the first attempt at this — a hand-rolled FastAPI/SQLite facts+notes app. This mem0 deployment is evaluating whether to replace it. As of this session: mem0 spike succeeded and this became the real deployment, but `bbmind` has **not** been decommissioned — no data has been migrated, and it's still running. Revisit once mem0 has some real-world track record.

## Known gaps / next steps

- Restart Claude Code and verify `/mcp` shows `mem0-bridge` connected; test `add_memory`/`search_memory` from a real session (not just the scripted stdio test done during setup).
- No NPM/SSL exposure yet (deliberate, see above).
- No backup strategy for the Postgres volume (`mem0-dev_postgres_db`) — worth adding to whatever backup routine covers the rest of this box's appdata.
- `bbmind`'s fate undecided.
