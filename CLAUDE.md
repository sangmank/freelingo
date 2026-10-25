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

**It is not running yet.** `DEPLOY.md` is the authority: it records what was done, what is
pending and why. Do not assume a live stack.

- Repo and deployment root: `/home/sangmank/projects/freelingo`, owned `sangmank:sangmank`.
- Docker Compose stack, project name `freelingo`. Compose itself is not yet installed on the host.
- Start only the four services the host can run:
  `docker compose -p freelingo up -d postgres redis backend frontend`. A bare `up -d` would also
  start `kokoro` and `whisper`, which reserve NVIDIA GPUs this machine does not have.
- Ports, loopback only, never `0.0.0.0`: `127.0.0.1:8420` frontend, `127.0.0.1:8421` backend. The
  backend port exists only so Caddy can reach `/ws/conversation`; Next.js does not proxy WebSocket
  upgrades. Everything else goes frontend to backend over the compose network. The `!override` tag
  on the frontend's `ports` in `docker-compose.override.yml` is load-bearing — without it Compose
  appends to upstream's list and leaves `0.0.0.0:3000` bound.
- **No provider keys are configured.** `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` are present but
  empty, on purpose, so the plumbing could be proven first. The tutor and voice features fail until
  they are filled in; that is expected, not a bug.
- Speech is configured for OpenAI TTS/STT rather than the bundled Kokoro and faster-whisper,
  because the host has no GPU.
- Caddy site file: canonical copy at `deploy/freelingo.caddyfile` in this repo, installed to
  `/etc/caddy/freelingo.caddyfile` and imported by one line in `/etc/caddy/Caddyfile`. Not yet
  installed — it waits on DNS for `freelingo.solsee.com`.
- Reload: `sudo caddy validate --config /etc/caddy/Caddyfile`, then `sudo systemctl reload caddy`.
  **Graceful reload only, never a restart**, and never the admin API on port 2019 — a `POST /load`
  there silently replaces the whole configuration and drops every other site on the host.
- Access lock-down: the app's own `ALLOW_REGISTRATION=false` in `.env`, set once the admin account
  exists. No edge gate, matching the other self-authenticating sites here (`ourtube`, `remod`,
  `nyquist`). Other sites use mTLS instead because they have no login of their own.
- Known deviation: `vm.overcommit_memory` stays at `0`, though upstream asks for `1`. Declined as a
  host-wide change. Redis still runs; the risk is a failed background save under memory pressure.

Read `/etc/caddy/README_AGENTS.md` before touching Caddy. Its reasoning is sound but two facts are
stale: the house IP is `98.207.230.73`, not the `98.210.42.172` it gives, and its port list is out
of date.

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
