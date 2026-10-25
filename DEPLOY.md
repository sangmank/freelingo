# DEPLOY.md — deploying this fork on the `sunnyvale` host

**Status of this document: the deployment is NOT running.** Steps 1–3 below were executed. Steps 4
onward are blocked on four decisions that are not mine to make (listed under "Blocked on" below) and
have **not** been run. They are written out so they can be run in order once those are resolved, but
nothing past step 3 has been verified on this machine.

No secrets appear in this file. The Anthropic API key goes only into `.env`, which `.gitignore`
already excludes.

## Host facts, verified 2026-10-25

- Host `sunnyvale`, Ubuntu 24.04.4, public IP `98.207.230.73`.
- Caddy v2 under systemd as `caddy:caddy`, one instance, config `/etc/caddy/Caddyfile`.
- The global block moves the listeners to `http_port 8480` / `https_port 8443`; the router NATs
  443 → 8443 and 80 → 8480. **Never write a port into a site address** — it desynchronises the ACME
  challenge wiring.
- Every site lives in its own `/etc/caddy/<name>.caddyfile`, imported by exactly one
  `import <name>.caddyfile` line in `/etc/caddy/Caddyfile`.
- Read `/etc/caddy/README_AGENTS.md` before touching anything in `/etc/caddy`. Note that its stated
  house IP (`98.210.42.172`) is **stale**; the current one is `98.207.230.73`, which is what every
  existing `*.solsee.com` subdomain resolves to.
- Apps live in `/home/sangmank/projects/<name>`, owned `sangmank:sangmank`.
- Every Caddy-fronted app on this host runs under **systemd**, not Docker. The three running
  containers (`trmnl-dash`, `med-track-dash`, `readlines-postgres`) are not behind Caddy and do not
  use Compose. This stack would be the first Compose-managed, Caddy-fronted app here.

### Caddy's two access patterns

The host uses exactly two, and the choice follows from whether the app authenticates itself:

- **mTLS at the edge**, each with its own CA — `climux`, `kickoff`, `council`, `pitchblende`. Used
  where the app has no login of its own, or where the capability is dangerous enough that the login
  path itself should not be reachable from the internet.
- **No edge gate, ordinary reverse proxy** — `nyquist`, `ourtube`, `remod`. Used where the app
  authenticates every request itself.

FreeLingo belongs in the second group: the backend enforces access on every request with JWT, and
public registration can be switched off in config (see below). So no edge gate is planned. This is
a judgement call, recorded here so it can be overridden.

### Ports

Loopback ports in use at survey time: 8000 nyquist, 8001 ourtube, 8080 council, 8147/8148/8152/8153
(uvicorn), 8410 remod, 8742 kickoff, 8766 taletail. Also 2019 caddy admin, 2300, 2400, 5432, 5037,
5177, 5199, 18080, 32400/32401.

Convention: web apps take a loopback port in the 8000–8800 range. Allocated for this stack:

- `127.0.0.1:8420` → frontend (container port 3000)
- `127.0.0.1:8421` → backend (container port 8000), needed only for the WebSocket

Both were free at survey time. Re-check with `ss -tlnp` before use.

Upstream's `docker-compose.yml` publishes only `3000:3000` — on all interfaces, which this host must
not do. Postgres and Redis publish no host ports, so there is no clash with the existing `5432`.

## Executed steps

### 1. Fork

```bash
gh repo fork ArtCC/freelingo --clone=false     # → https://github.com/sangmank/freelingo
```

### 2. Clone and wire up `upstream`

```bash
cd /home/sangmank/projects
git clone https://github.com/sangmank/freelingo.git freelingo
cd freelingo
git remote add upstream https://github.com/ArtCC/freelingo.git
```

### 3. Branch for local setup

```bash
git checkout -b local/setup
```

## Blocked on

Each of these stops the deployment. None was decided unilaterally.

1. **No Anthropic API key on this machine.** `ANTHROPIC_API_KEY` is not set in the environment, and
   no key was provisioned for this app. A key that exists for Claude Code's own use is not
   automatically available to a third-party app — that reuse needs an explicit decision.
2. **Docker Compose is not installed.** Neither `docker compose` nor `docker-compose` exists; only
   the `docker-trust` CLI plugin is present. Installing it means
   `sudo apt install docker-compose-v2` (candidate 2.40.3), a system package install.
3. **DNS does not resolve.** `freelingo.solsee.com` is NXDOMAIN. Without it Let's Encrypt cannot
   issue, so the site cannot come up. Needed: an **A record `freelingo.solsee.com` →
   `98.207.230.73`, DNS only (grey cloud, not Cloudflare-proxied)**, matching how every other
   `*.solsee.com` subdomain is set. The apex `solsee.com` *is* proxied; do not copy the apex.
4. **`vm.overcommit_memory=1` is required** by upstream for safe Redis background saves. That is a
   host-wide sysctl.

And one deviation that needs a decision rather than blocking outright:

5. **This host has no NVIDIA GPU.** `nvidia-smi` is absent and Docker has no `nvidia` runtime, but
   upstream's Kokoro and Whisper services both request GPU reservations. CPU images and a smaller
   Whisper model are needed. Upstream states plainly that *"exact upstream tags are not a stable
   FreeLingo contract"* (`specs/docker.instructions.md`), so the tags below are a guess that must be
   checked. Expect voice to be slow on CPU.

## Remaining steps — NOT YET RUN

### 4. Install Compose (gated on decision 2)

```bash
sudo apt install docker-compose-v2
```

### 5. Configure `.env` (gated on decision 1)

```bash
cd /home/sangmank/projects/freelingo
cp .env.example .env
chmod 600 .env
```

Then edit `.env`. Values that matter here:

- `LLM_PROVIDER=anthropic`, and set `ANTHROPIC_API_KEY` to the key. Nothing else; leave
  `OPENAI_API_KEY` empty.
- `TTS_PROVIDER=local` and `STT_PROVIDER=local` — the bundled Kokoro and faster-whisper, so that no
  OpenAI key is needed.
- `STT_MODEL=small` for CPU.
- Replace every `CHANGE_ME_*` placeholder, including `POSTGRES_PASSWORD`, `REDIS_PASSWORD` and the
  JWT signing secret (`openssl rand -hex 32`).
- `DATA_PATH` to wherever the Postgres and Redis volumes should live.
- Leave `ALLOW_REGISTRATION=true` and `FIRST_USER_IS_ADMIN=true` **for now** — step 8 closes
  registration after the admin account exists.
- Stripe/quota/trial settings stay off.

`.env` is already covered by `.gitignore`. Never commit it.

### 6. Deployment-only Compose override (gated on decision 5)

Write `docker-compose.override.yml`, which upstream does not ship. Keeping these out of
`docker-compose.yml` is what makes the monthly upstream rebase clean.

```yaml
# docker-compose.override.yml — this deployment only, never upstreamed.
services:
  frontend:
    ports:
      - "127.0.0.1:8420:3000"

  backend:
    ports:
      - "127.0.0.1:8421:8000"

  # No NVIDIA GPU on this host: CPU images, GPU reservations removed.
  kokoro:
    image: ghcr.io/remsky/kokoro-fastapi-cpu:latest
    deploy: !reset null

  whisper:
    image: onerahmet/openai-whisper-asr-webservice:latest
    environment:
      - ASR_MODEL=small
      - ASR_ENGINE=faster_whisper
    deploy: !reset null
```

Two things to verify when this is first run, neither of which could be checked without Compose
installed: that the CPU image tags above actually exist, and that `!reset` genuinely clears the
inherited GPU reservation (it needs Compose 2.24+; 2.40 is the apt candidate, so it should).

The backend port is published on loopback only. Note the caveat `/etc/caddy/README_AGENTS.md`
raises: any local user on this host can reach a loopback port directly. That is already true of
nyquist, council, remod and kickoff, so it is the house norm rather than a new exposure — but it is
the reason the backend must not be published on `0.0.0.0`.

### 7. Bring the stack up

```bash
cd /home/sangmank/projects/freelingo
docker compose -p freelingo up -d
docker compose -p freelingo ps
curl -sS http://127.0.0.1:8421/health
curl -sSI http://127.0.0.1:8420/
```

Container names are already unique in upstream's compose (`freelingo-postgres`, `freelingo-redis`,
`freelingo-backend`, `freelingo-frontend`, `freelingo-kokoro`, `freelingo-whisper`), so nothing
collides with the existing `readlines-postgres`.

### 8. Create the admin account, then close registration

This is the whole access story, so do not skip the second half.

1. With the stack up and `ALLOW_REGISTRATION=true`, register once at
   `http://127.0.0.1:8420/`. The first user becomes admin
   (`backend/app/routers/auth.py` — `role = "admin" if user_count == 0 and FIRST_USER_IS_ADMIN`).
2. Confirm the account is admin.
3. Set `ALLOW_REGISTRATION=false` in `.env`.
4. `docker compose -p freelingo up -d backend` to pick it up.
5. Verify a stranger is refused: a `POST` to the register endpoint must fail, and
   `GET /api/config` must report `"allow_registration": false`.

The flag is enforced server-side at `backend/app/routers/auth.py:64` and covered by upstream tests
in `backend/tests/test_auth.py`, so this is a real gate, not a hidden frontend control.

### 9. Caddy site (gated on decision 3 — DNS must resolve first)

Write `/etc/caddy/freelingo.caddyfile`:

```caddyfile
# /etc/caddy/freelingo.caddyfile — imported from /etc/caddy/Caddyfile
# (see /etc/caddy/README_AGENTS.md)
#
# No port in the site address, and no `tls` directive: the global block moves the
# listeners to 8443/8480 and rewires the ACME challenges to match, and staying out of
# `tls` leaves the climux.solsee.com client-certificate policy untouched.

freelingo.solsee.com {
	encode zstd gzip

	# FreeLingo authenticates every request itself: the backend enforces access on every
	# call and the frontend's route guards are navigation aids only (upstream AGENTS.md).
	# Public sign-up is closed in the app's own config with ALLOW_REGISTRATION=false, so
	# there is no edge-level gate standing in for the app. Same shape as ourtube and remod.

	# Next.js does not proxy WebSocket upgrades, so /ws/* must reach the backend directly
	# rather than going through the frontend (upstream README, "Reverse proxy requirement").
	handle /ws/* {
		reverse_proxy 127.0.0.1:8421
	}

	handle {
		reverse_proxy 127.0.0.1:8420 {
			header_up X-Real-IP {remote_host}
			header_up X-Forwarded-For {remote_host}
			header_up X-Forwarded-Proto {scheme}
		}
	}
}
```

Add exactly one line to `/etc/caddy/Caddyfile`:

```
import freelingo.caddyfile
```

Then, in this order:

```bash
sudo caddy fmt --overwrite /etc/caddy/freelingo.caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile      # always, before any reload
sudo systemctl reload caddy                            # graceful; never restart
```

Never use the Caddy admin API on `127.0.0.1:2019` to add the site: a `POST /load` replaces the
entire configuration and silently removes every other site.

This block declares no `tls` directive, so it cannot perturb the mTLS connection policies of the
other sites — `README_AGENTS.md` verified exactly this case. No policy check is needed. If a `tls`
directive is ever added here, run the connection-policy check in that README first.

### 10. Confirm nothing else broke

Before reloading, record the current responses; after reloading, compare:

```bash
for h in nyquist climux kickoff council ourtube pitchblende remod; do
  printf '%-14s %s\n' "$h" "$(curl -sS -o /dev/null -w '%{http_code}' --max-time 10 "https://$h.solsee.com/")"
done
```

`climux`, `kickoff`, `council` and `pitchblende` require a client certificate, so a bare curl fails
against them — what matters is that each behaves the *same* before and after. If anything changed,
remove the `import freelingo.caddyfile` line, `caddy validate`, reload, and stop.

### 11. Smoke test

- `https://freelingo.solsee.com` loads with a valid Let's Encrypt certificate.
- Sign-up is refused for a stranger.
- Admin login works.
- One text exchange with the tutor returns a reply (exercises the Anthropic key).
- One voice exchange works end to end over `/ws/conversation` (exercises Whisper and Kokoro, and
  the Caddy WebSocket route). On CPU, expect this to be slow.

## Rollback

- Caddy: delete the `import freelingo.caddyfile` line, `caddy validate`, `systemctl reload caddy`.
- Stack: `docker compose -p freelingo down`. Add `-v` only to discard the database as well.
