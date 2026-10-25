# CLAUDE.md — sangmank/freelingo

A fork of [ArtCC/freelingo](https://github.com/ArtCC/freelingo), self-hosted for one household.

## Read these first

- `AGENTS.md` — upstream's invariants and documentation workflow. **Binding on this fork.** Do not
  weaken the auth, refresh-token or storage rules it protects.
- `specs/` — upstream's specifications, the source of truth for current behavior. Do not copy them
  into this fork's docs; link to them.
- `SPEC.md` — this fork's delta spec: what we intend to add and why. Everything in it is draft and
  nothing in it is built yet.
- `DEPLOY.md` — how this machine's deployment works, and how to rebuild it.
- `CHANGELOG.md` — upstream's release history. Not a place to record fork-local changes.

## The upstream-sync rule

This fork tracks upstream and is rebased on it **monthly**. Everything else follows from that:

- Put new behavior in new modules, services and config flags — not in edits to existing upstream
  files.
- When an upstream file genuinely must change, make the edit the smallest possible hook and keep the
  real logic in a new file.
- Keep new features off by default behind a flag, so an unconfigured deployment matches upstream.
- Deployment-only configuration belongs in files upstream does not ship — `docker-compose.override.yml`,
  `.env` — never in `docker-compose.yml`.
- `upstream` is configured as a remote. Rebase, do not merge, so the fork stays a readable diff.
- Never open a PR against upstream from this fork without being asked.

## This machine's deployment

**It is not running yet.** `DEPLOY.md` records what was done, what is blocked and on what. Do not
assume a live stack. The summary below is the intended shape.

- Repo and deployment root: `/home/sangmank/projects/freelingo`, owned `sangmank:sangmank`.
- Runs as a Docker Compose stack, project name `freelingo`.
- Ports, loopback only, never `0.0.0.0`: `127.0.0.1:8420` frontend, `127.0.0.1:8421` backend. The
  backend port exists only so Caddy can reach the WebSocket; everything else goes through the
  frontend.
- Caddy site file: `/etc/caddy/freelingo.caddyfile`, imported by one line in `/etc/caddy/Caddyfile`.
  It routes `/ws/*` to the backend, because Next.js does not proxy WebSocket upgrades.
- Reload: `sudo caddy validate --config /etc/caddy/Caddyfile` then
  `sudo systemctl reload caddy`. **Graceful reload only, never a restart**, and never the admin API
  on port 2019 — a `POST /load` there silently replaces the whole configuration and drops every
  other site on the host.
- Access lock-down: the app's own `ALLOW_REGISTRATION=false` in `.env`, set after the admin account
  exists. There is no edge-level gate, matching the other self-authenticating sites on this host
  (`ourtube`, `remod`, `nyquist`). Other sites here use mTLS instead because they have no login of
  their own.
- Host constraints worth knowing: no NVIDIA GPU, so Kokoro and Whisper run from CPU images and voice
  is slow; Redis wants `vm.overcommit_memory=1` on the host.

Read `/etc/caddy/README_AGENTS.md` before touching Caddy. Its house IP is stale — the current one is
`98.207.230.73`.

## Shared state on this host

This machine runs other people's services, including a music server at `nyquist.solsee.com`. Ask
before anything that reaches outside this app: editing the global Caddy config or another site's
file, installing system packages, changing sysctls, opening firewall ports, or stopping any existing
container or service. Do not touch `/home/sangmank/projects/nyquist*`.

## Secrets

The Anthropic API key lives only in `.env`, which `.gitignore` already excludes. Do not commit it,
print it, paste it into a doc, or copy it into another file. The same goes for the Postgres, Redis
and JWT secrets.

## Licence

AGPL-3.0, inherited from upstream. This deployment serves one household. The fork stays public
because of the AGPL's network-use provision.
