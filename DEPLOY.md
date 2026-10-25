# DEPLOY.md — deploying this fork on the `sunnyvale` host

**Status: the stack is NOT running and Caddy has NOT been touched.** Configuration is complete and
validated; two things block the start, neither of them a decision:

- **Compose is still not installed.** The install was approved, but `sudo` on this host requires a
  password and this session has no terminal, so it could not be run unattended. One command, below.
- **`freelingo.solsee.com` does not resolve yet.** The DNS record is being added by the owner.
  Until `dig` returns `98.207.230.73`, the Caddy site must not be installed: Let's Encrypt would
  fail and the failure would be logged against the whole config.

No secrets appear in this file. The API key slots in `.env` are deliberately empty — see "LLM and
speech keys" below.

## Host facts, verified 2026-10-25

- Host `sunnyvale`, Ubuntu 24.04.4, public IP `98.207.230.73`.
- Caddy v2 under systemd as `caddy:caddy`, one instance, config `/etc/caddy/Caddyfile`.
- The global block moves the listeners to `http_port 8480` / `https_port 8443`; the router NATs
  443 to 8443 and 80 to 8480. **Never write a port into a site address** — it desynchronises the
  ACME challenge wiring.
- Every site lives in its own `/etc/caddy/<name>.caddyfile`, imported by exactly one
  `import <name>.caddyfile` line in `/etc/caddy/Caddyfile`.
- Read `/etc/caddy/README_AGENTS.md` before touching anything in `/etc/caddy`. **Two of its facts
  are stale:** it gives the house IP as `98.210.42.172` (actually `98.207.230.73`) and its port
  list predates several services. Its reasoning is still sound and worth following.
- Apps live in `/home/sangmank/projects/<name>`, owned `sangmank:sangmank`.
- Every Caddy-fronted app here runs under **systemd**, not Docker. The three running containers
  (`trmnl-dash`, `med-track-dash`, `readlines-postgres`) are not behind Caddy and do not use
  Compose. This stack is the first Compose-managed, Caddy-fronted app on the host.
- **No NVIDIA GPU**: `nvidia-smi` is absent and Docker has no `nvidia` runtime.

### Caddy's two access patterns

The host uses exactly two, and the choice follows from whether the app authenticates itself:

- **mTLS at the edge**, each with its own CA — `climux`, `kickoff`, `council`, `pitchblende`. Used
  where the app has no login of its own, or where the capability is dangerous enough that the login
  path itself should not be reachable from the internet.
- **No edge gate, ordinary reverse proxy** — `nyquist`, `ourtube`, `remod`. Used where the app
  authenticates every request itself.

FreeLingo belongs in the second group: the backend enforces access on every request, and public
sign-up is closed in the app's own config. So no edge gate.

### Ports

Loopback ports in use at survey time: 8000 nyquist, 8001 ourtube, 8080 council, 8147/8148/8152/8153
(uvicorn), 8410 remod, 8742 kickoff, 8766 taletail. Also 2019 caddy admin, 2300, 2400, 5432, 5037,
5177, 5199, 18080, 32400/32401.

Convention: web apps take a loopback port in the 8000-8800 range. Allocated here:

- `127.0.0.1:8420` — frontend (container port 3000)
- `127.0.0.1:8421` — backend (container port 8000), for the WebSocket only

Postgres and Redis publish no host ports, so there is no clash with the existing `5432`.

## LLM and speech keys — deliberately absent

This deployment is being brought up **without any provider keys**, to prove the plumbing first:

- `LLM_PROVIDER=anthropic` with `ANTHROPIC_API_KEY=` **empty**.
- `TTS_PROVIDER=openai` and `STT_PROVIDER=openai` with `OPENAI_API_KEY=` **empty**.

Everything that does not call a provider works. The tutor text exchange and the voice exchange will
fail until the keys are filled in — that is expected, not a defect. Add the keys to `.env` and run
`docker compose -p freelingo up -d backend` to pick them up.

OpenAI speech was chosen over the bundled local services because this host has no GPU. That also
means **the `kokoro` and `whisper` services are never started.** They are not deleted from
upstream's `docker-compose.yml` — nothing `depends_on` them, and their only other role is supplying
the default `TTS_BASE_URL` / `STT_BASE_URL`, which are unused when the providers are OpenAI. They
are simply left out of the `up` command. This avoids both the GPU reservations and the guessed CPU
image tags that upstream warns "are not a stable FreeLingo contract".

Note for the SPEC direction on multilingual TTS: Kokoro's bundled voices are English-only, so local
TTS would not have served a non-English target language regardless of GPU.

### If anyone revisits local speech

Two findings worth not rediscovering, both checked against the registries on 2026-10-25:

- Upstream's compose comment says "CPU-only host: use `:cpu-latest`". **No such tag exists** on
  either Kokoro repo. The GPU repo `ghcr.io/remsky/kokoro-fastapi-gpu` publishes only `latest`,
  `latest-cu126`, `latest-cu128` (plus arch variants). The CPU build is a *separate repo*,
  `ghcr.io/remsky/kokoro-fastapi-cpu`, with tags `latest`, `latest-slim`, `latest-amd64`,
  `latest-arm64`. So the correct CPU image is `ghcr.io/remsky/kokoro-fastapi-cpu:latest`. This is
  consistent with upstream's own disclaimer that CPU tags "are not a stable FreeLingo contract".
- Whisper's CPU image is straightforward: `onerahmet/openai-whisper-asr-webservice:latest` (the
  `-gpu` suffix marks the GPU builds), with `ASR_MODEL=base` or `small`.

Dropping the GPU reservation from an override file would need Compose's `!reset` or `!override`
tag; that path was never exercised here and is unverified.

And the reason not to bother: Kokoro's voices are English-only, so local TTS cannot serve the
fork's multilingual-TTS direction (SPEC item 3) whatever the hardware. A different engine is
needed, not a different image tag.

## Known deviations from upstream guidance

- **`vm.overcommit_memory` is left at `0`.** Upstream asks for `1`. This was explicitly declined as
  a host-wide change. It is not required for the stack to start — Redis runs and logs a warning.
  The exposure is that a background RDB save can fail under memory pressure and Redis may lose data
  on restart. Here Redis holds cache and session state while Postgres is the system of record, so
  the blast radius is sessions, not learner data.

## Two traps, both the same shape

Each of these is an edit that **succeeds, reports success, and silently leaves the wrong value in
place**. Neither surfaces as an error. Check both explicitly on any rerun.

### `.env.example` has duplicate keys

`LLM_PROVIDER=ollama`, `TTS_PROVIDER=local` and `STT_PROVIDER=local` each appear **twice** in
`.env.example`: once as the real assignment, once inside an explanatory comment just above it
(lines 46/51, 74/71, 84/81). A `sed s/LLM_PROVIDER=ollama/.../` edits whichever it reaches first
and can leave the app pointed at Ollama while the diff looks right.

Anchor every substitution to the start of a line, and assert exactly one match before writing.
Afterwards verify the values that matter:

```bash
grep -E '^(LLM_PROVIDER|TTS_PROVIDER|STT_PROVIDER)=' .env
```

### Compose appends port lists, it does not replace them

`docker-compose.override.yml` remaps the frontend to loopback, but Compose **merges `ports` by
appending**. Without the `!override` tag the result is upstream's `0.0.0.0:3000` *and* the loopback
mapping — the stack comes up, the app works, and the public binding this host must never have is
quietly live.

This is the single security-relevant assumption in the setup that has not been exercised, because
Compose was not installed when the override was written. **Verify it before the first `up`:**

```bash
docker compose -p freelingo config | grep -A4 'ports:'
```

Every published port must begin `127.0.0.1:`. If any entry lacks it, stop — do not start the stack
and do not add the Caddy site.

## Executed steps

### 1. Fork, clone, remotes

```bash
gh repo fork ArtCC/freelingo --clone=false     # -> https://github.com/sangmank/freelingo
cd /home/sangmank/projects
git clone https://github.com/sangmank/freelingo.git freelingo
cd freelingo
git remote add upstream https://github.com/ArtCC/freelingo.git
git checkout -b local/setup
```

### 2. `.env`

```bash
cd /home/sangmank/projects/freelingo
umask 077
cp .env.example .env
chmod 600 .env
mkdir -p data
```

Then set the following, **anchoring each edit to the start of a line** — three of these strings also
occur inside comments in `.env.example`, so a naive search-and-replace hits the wrong occurrence:

- `LLM_PROVIDER=anthropic`, `ANTHROPIC_API_KEY=` (empty for now)
- `TTS_PROVIDER=openai`, `STT_PROVIDER=openai`, `OPENAI_API_KEY=` (empty for now)
- `POSTGRES_PASSWORD`, `REDIS_PASSWORD` from `openssl rand -hex 24`
- `SECRET_KEY` from `openssl rand -hex 32`
- `DATA_PATH=/home/sangmank/projects/freelingo/data` (upstream's default `/opt/docker/freelingo`
  would need root; `/data/` is already in `.gitignore`)
- `CORS_ORIGINS=["https://freelingo.solsee.com"]`
- `COOKIE_SECURE=true`, `APP_BASE_URL=https://freelingo.solsee.com`
- Left as-is: `ALLOW_REGISTRATION=true` and `FIRST_USER_IS_ADMIN=true` until step 6 closes sign-up;
  `STRIPE_ENABLED=false`, already upstream's default.

`.env` is mode 600 and matched by `.gitignore`. Never commit it.

### 3. Compose override

`docker-compose.override.yml` in this repo republishes both services on loopback. Compose loads it
automatically. Note the `!override` tag on the frontend's `ports`: without it Compose **appends** to
upstream's list and `0.0.0.0:3000` stays bound, which is exactly what must not happen on this host.
Confirm with `docker compose -p freelingo config` before the first `up`.

### 4. Caddy site file — written and validated, NOT installed

The canonical copy is `deploy/freelingo.caddyfile` in this repo. It routes `/ws/*` to the backend,
because Next.js does not proxy WebSocket upgrades, and everything else to the frontend.

Validated against a scratch copy of the host's full config with the import line added:

- `caddy validate` returned `Valid configuration`.
- The four mTLS connection policies (`pitchblende`, `kickoff`, `council`, `climux`, each
  `require_and_verify`, with the bare catch-all last) are **byte-identical** with and without this
  site, confirming it cannot weaken them. This block declares no `tls` directive, which is the case
  `README_AGENTS.md` verified.

Baseline responses recorded before any change, to compare against afterwards: `nyquist` 303,
`ourtube` 307, `remod` 303, and `climux` / `kickoff` / `council` / `pitchblende` all correctly
refusing a connection that presents no client certificate.

## Remaining steps — NOT YET RUN

### 5. Install Compose and start the stack

Needs an interactive terminal for the sudo password:

```bash
sudo apt install docker-compose-v2
cd /home/sangmank/projects/freelingo
docker compose -p freelingo config | grep -A3 'ports:'      # confirm loopback only, no 0.0.0.0
docker compose -p freelingo up -d postgres redis backend frontend
docker compose -p freelingo ps
curl -sS http://127.0.0.1:8421/health
curl -sSI http://127.0.0.1:8420/
```

Name the four services explicitly — a bare `up -d` would also start `kokoro` and `whisper`, which
request GPUs this host does not have.

Container names are already unique upstream (`freelingo-postgres`, `freelingo-redis`,
`freelingo-backend`, `freelingo-frontend`), so nothing collides with `readlines-postgres`.

### 6. Create the admin account, then close registration

This is the whole access story; do not stop halfway.

1. With the stack up and `ALLOW_REGISTRATION=true`, register once at `http://127.0.0.1:8420/`.
   The first user becomes admin (`backend/app/routers/auth.py` — `role = "admin" if user_count == 0
   and FIRST_USER_IS_ADMIN`).
2. Confirm the account is admin.
3. Set `ALLOW_REGISTRATION=false` in `.env`.
4. `docker compose -p freelingo up -d backend`.
5. Verify a stranger is refused: a `POST` to the register endpoint must fail, and `GET /api/config`
   must report `"allow_registration": false`.

Enforced server-side at `backend/app/routers/auth.py:64`, with upstream tests in
`backend/tests/test_auth.py`. It is a real gate, not a frontend control.

### 7. Caddy — only once DNS resolves

```bash
dig +short freelingo.solsee.com A      # must print 98.207.230.73 before going further
```

Then:

```bash
sudo cp /home/sangmank/projects/freelingo/deploy/freelingo.caddyfile /etc/caddy/
echo 'import freelingo.caddyfile' | sudo tee -a /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile      # always, before any reload
sudo systemctl reload caddy                            # graceful; never restart
```

Never use the Caddy admin API on `127.0.0.1:2019`: a `POST /load` there replaces the entire
configuration and silently drops every other site.

Then re-check the other sites against the baseline above:

```bash
for h in nyquist climux kickoff council ourtube pitchblende remod; do
  printf '%-13s %s\n' "$h" "$(curl -sS -o /dev/null -w '%{http_code}' --max-time 10 "https://$h.solsee.com/")"
done
```

The four mTLS sites fail a bare curl by design — what matters is that each behaves the *same* as
before. On any change: remove the import line, `caddy validate`, reload, stop and report.

### 8. Smoke test

Expected to pass: the site loads over HTTPS with a valid Let's Encrypt certificate; sign-up is
refused for a stranger; admin login works.

Expected to fail until keys are added: the tutor text exchange (no `ANTHROPIC_API_KEY`) and the
voice exchange (no `OPENAI_API_KEY` for TTS and STT).

## Rollback

- Caddy: delete the `import freelingo.caddyfile` line, `caddy validate`, `systemctl reload caddy`.
- Stack: `docker compose -p freelingo down`. Add `-v` only to discard the database too.
