# JobTracker Infra (Front Door :7014)

Hey there! Welcome to the front porch of JobTracker.

Think of this folder as the single doorway to the whole shop. Caddy stands in the opening and points everyone to the right room: your browser knocks once on `:7014`, and this folder decides whether the knock goes to the application locker (`:7012`) or to the login desk (`:7002`).

Best of all: two scripts, two buttons. Press one to open, press one to close — and nothing else in your machine even notices.

---

## What Does This Helper Do?

- Front Door (`Caddyfile`): Routes `/api/*` → `127.0.0.1:7012`, `/auth/*` → `127.0.0.1:7002` (with `/auth` stripped off, because the frozen PHP auth wants its paths exactly as they were), and answers `/health` with a plain `ok`.
- Opening Button (`scripts/start.sh`): Builds the backend, waits up to 60 seconds for it to feel healthy, then checks four endpoints so you don't have to guess. It even creates `.env` for you if you forgot step 1.
- Closing Button (`scripts/stop.sh`): Brings *this stack only* down. Your auth server, Postgres, and n8n keep right on running.
- Host Whisperer: Postgres isn't in this stack — it's already alive on `:5432`. The compose file quietly points `DB_HOST` and `AUTH_URL` at `host.docker.internal` so the container can reach them on your machine.
- Patient Doorman (`healthcheck`): Pokes `/health` every 10 seconds and only lets Caddy in once the backend answers. Six misses in a row and the backend is marked unhealthy — Caddy stays shut out.
- Log Keeper: Both services write JSON logs to stdout, so `docker compose logs` stays readable.

---

## Getting Started (Super Easy!)

You don't need a computer science degree to get this running. Just follow these steps:

### 1. Grab your config sheet
If `.env` isn't there yet, `start.sh` copies the template for you:
```bash
cp .env.example .env
```
(It's still worth opening it once — your Postgres password lives in there.)

### 2. Check your tools
This folder only needs Docker with the compose plugin:
```bash
docker --version
docker compose version
```

### 3. Turn on the lights!
```bash
bash infra/scripts/start.sh
```
It runs `up -d --build`, waits for the backend to go green, then verifies all four routes for you. When it finishes clean — no failed checks — you're open for business.

### 4. Close up shop
```bash
bash infra/scripts/stop.sh
```
Only the backend and Caddy go dark. Everything else stays as it was.

That is it! Your front door is now answering at http://localhost:7014.

---

## How to Tell If It's Working

Ring the doorbell from a terminal:
```bash
curl http://localhost:7014/health
```
If you get `ok`, the door is open.

Follow the knock through to the backend:
```bash
curl http://localhost:7014/api/health
```
If you get `{"ok":true}`, the proxy really is reaching port 7012.

Check that it can find the login desk too:
```bash
curl http://localhost:7014/auth/api/me
```
A `401` here is *good news* — it means the proxy reached auth and simply found no ticket.

Not using Docker? The backend still answers directly:
```bash
curl http://localhost:7012/health
```

---

## What's Inside the Box?

```text
infra/
├── Caddyfile                  # The front door's direction board for :7014
├── docker-compose.stack.yml   # backend-api (multi-stage) + caddy:2-alpine
├── scripts/
│   ├── start.sh               # Build, wait for health, verify 4 endpoints
│   └── stop.sh                # Down — this stack only, nothing else touched
└── README.md                  # This file
```

---

## Golden House Rules

1. Share the Database, Don't Ship It: Postgres is intentionally *not* part of this stack. It reuses the container already running on `:5432`, and `DB_HOST` is overridden to `host.docker.internal` the moment the container wakes up.
2. Caddy Uses Host Networking On Purpose: `network_mode: host` makes the `127.0.0.1:7012/7002` addresses in the Caddyfile literally true — no guessing container DNS names.
3. Always Comes Back: `restart: always` (not `unless-stopped`) is a deliberate call — boot the PC and the containers come back on their own. Forgetting is allowed.
4. Mind the Other Doors: `start.sh` and `stop.sh` only ever touch this stack. Ports `443`, `8443`, `9443`, and `10000` already belong to other services — hands off.
5. One Knock at a Time: Don't run `npm run dev` from the backend folder while this stack is up — they will argue over port 7012.

### Opening the shop to the outside (optional)
```bash
tailscale serve --bg --https=7443 http://127.0.0.1:7014
tailscale funnel status
```

Enjoy building!
