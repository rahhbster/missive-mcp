---
title: Backlog
date: 2026-08-15
type: backlog
---

# Backlog

Open work items and known gaps. Newest context at the top of each section.

## Current State (as of 2026-08-15)

The repo is cloned and running locally in **stdio mode**, registered with
Claude Desktop and verified end to end.

- Node **26.7.0** (Homebrew, `/opt/homebrew/bin/node`); deps installed, `npm run build` clean
- Registered in `~/Library/Application Support/Claude/claude_desktop_config.json`
  under `mcpServers.missive`, pointing at `/opt/homebrew/bin/node` +
  `dist/index.js`, with `MISSIVE_API_TOKEN` in the entry's `env` block
- Verified by driving the server over stdio with that exact config:
  handshake OK, **19 tools** registered, `list_organizations` returned
  `Connected Well` and `Merrill Family` with `isError: false`
- Dependencies patched — `npm audit` reports 0 vulnerabilities
- A config backup exists at `claude_desktop_config.json.bak`

Branch is `development` (the repo's default; there is no `main`).

## Open Items

### 1. README omits `draft_reply`

The server registers **19** tools, but the README's tables document 18.
`draft_reply` is implemented (`src/tools/drafts.ts`) and registered, yet
missing from the Drafts table. Doc drift from the February tool work.

**Fix:** add a `draft_reply` row to the Drafts table in `README.md`.

### 2. Docker image is never built by CI

`.github/workflows/docker.yml` triggers on pushes to `main` and on PRs to
`main`. This repo's only branch is `development`, so the workflow has
**never run** and no image exists at `ghcr.io`.

**Fix:** either retarget the workflow to `development`, or rename the
default branch. Until then, any deployment must build the image locally.

### 3. Remote mode: open dynamic client registration

`src/server.ts` mounts `mcpAuthRouter`, which exposes `/register` for
unauthenticated dynamic client registration, plus the `/authorize-form`
token-entry page. Anyone who reaches the URL can register a client and
grow `clients.json` without bound.

This is **not** a path to the operator's mail — the server is multi-tenant
and every tool call resolves a PAT the caller supplied themselves. But it
is unauthenticated write-to-disk on a public endpoint.

**Mitigation if hosted publicly:** IP allowlist or basic auth in the
reverse proxy in front of `/register` and `/authorize-form`.

### 4. Remote mode: operational constraints

Carried forward for whenever remote deployment happens:

- **`DATA_DIR` needs a persistent volume.** OAuth state and encrypted PATs
  are JSON files under `./data` by default. Without a mount, every restart
  logs all users out.
- **`ENCRYPTION_KEY` must be preserved.** It decrypts stored PATs
  (AES-256-GCM). Lose it and every user must re-authorize.
- **Single instance only.** `src/auth/storage.ts` keeps state in an
  in-memory map flushed to disk with no locking, so two replicas would
  corrupt each other. Do not scale horizontally.
- Remote mode is newer than stdio — it landed in a three-commit burst on
  2026-02-17 plus a shutdown fix on 02-18. Less exercised; expect rough
  edges.

### 5. Deployment path not yet chosen

Two options were evaluated, neither started:

- **Tailscale** — `tailscale serve --bg --https=443 localhost:3000` gives a
  real Let's Encrypt cert on the MagicDNS name with zero public exposure.
  Preferred when clients are the user's own devices. Does **not** work for
  claude.ai web/mobile, whose connections originate from Anthropic's
  servers, not the tailnet.
- **Lightsail** — public host + domain + Caddy for TLS. Required if
  claude.ai web/mobile must connect. Inherits items 3 and 4.

A useful intermediate check before either: run remote mode on localhost,
where the MCP HTTPS requirement is waived.

```bash
ENCRYPTION_KEY="$(openssl rand -hex 32)" BASE_URL="http://localhost:3000" npm run remote
```

## Environment Notes

- `dist/` is gitignored. A `git clean -xdf` deletes the build and breaks
  the Desktop integration until `npm run build` is re-run.
- The Missive token lives in plaintext in the Desktop config — normal for
  MCP stdio servers, but keep that file out of version control.
- `package.json` now declares `node >=20.0.0`. `@hono/node-server` 2.x
  (pulled in by the security update) requires Node 20+, so the previous
  `>=18` floor would fail to install. The Dockerfile pins `node:22-slim`
  and is unaffected.
