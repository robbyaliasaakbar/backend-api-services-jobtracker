# JobTracker Backend (Lamaran API :7012 + Front Door :7014)

Hey there! Welcome to the engine room of JobTracker.

Think of this server as the tidy filing clerk behind a long counter. It takes job applications from your frontend, checks that every entry makes sense (company name? date format? a status you actually picked?), asks the Central Auth server to stamp your ticket before letting you near private data, and files everything into PostgreSQL so the dashboard looks sharp.

Best of all: one door in front (:7014), so the browser only ever needs to remember a single address.

---

## What Does This Helper Do?

- Application Locker (`/api/lamaran`): Lists, creates, updates, and deletes your job applications — `GET /`, `POST /`, `PUT /:id`, `DELETE /:id`.
- Fussy Librarian: Rejects an application without a company or position, insists on `yyyy-mm-dd` dates, and quietly adds `https://` to links you forgot to prefix.
- Status Keeper: Tracks each application through `baru` → `interview-hr` → `technical-test` → `interview-user` → `offering` → `diterima` / `ditolak`. Anything outside that list gets turned away.
- Friendly Bouncer: Asks our Central Auth server (`:7002`) to verify your Bearer ticket on every private route — it only ever *reads* from auth, never writes.
- Front Door (Caddy `:7014`): One reverse proxy that routes `/api/*` to this backend and `/auth/*` to Central Auth, so the frontend never juggles ports.
- Caretaker: Remembers which email owns which application, thanks to the token's owner coming straight from auth.

---

## Getting Started (Super Easy!)

You don't need a computer science degree to get this running. Just follow these steps:

### 1. Grab your config sheet
Duplicate the template file into `.env`:
```bash
cp .env.example .env
```
Fill in your Postgres user/password (Cara A), or drop in a full `DATABASE_URL` (Cara B, which wins). The auth URL and allowed CORS origins already point at the right places — leave them alone unless you moved something.

### 2. Set up the tools
If you're running directly on your machine:
```bash
npm install
```
(You'll need Node 22 or newer.)

### 3. Build the shelves
Create the `lamaran` table in your `jobtracker_v2` database:
```bash
npm run migrate
```

### 4. Turn on the lights!

Option A — Directly with Node:
```bash
npm run dev     # auto-restarts while you code
# or
npm start       # plain and steady
```

Option B — Cozy inside Docker (Set & Forget):
```bash
bash infra/scripts/start.sh
```
That brings up the backend *and* the Caddy front door, waits until they're healthy, then verifies all four endpoints for you. Shut it down with `bash infra/scripts/stop.sh` — and it won't touch your other containers.

That is it! Your API is now serving requests at http://localhost:7012 (or through the front door at http://localhost:7014).

---

## How to Tell If It's Working

Pop open a terminal or your browser and check the health buzzer:
```bash
curl http://localhost:7012/health
```
If you get `{"ok":true}`, you are good to go.

Want to check the front door too?
```bash
curl http://localhost:7014/health
```

To run a quick self-check test suite:
```bash
npm test
```

And a lint pass to keep the code honest:
```bash
npm run lint
```

---

## What's Inside the Box?

```text
.
├── src/
│   ├── server.js            # Wakes up the shop and starts listening
│   ├── app.js               # The main counter answering all API requests
│   ├── routes/lamaran.js    # Every /api/lamaran route lives here
│   ├── auth.js              # The bouncer checking tickets with Central Auth
│   ├── validate.js          # Rules for company, date, status & link
│   ├── config.js            # Reads your .env settings without extra fluff
│   ├── db.js                # Connects to your PostgreSQL storage
│   └── pgtypes.js           # Keeps Postgres types friendly for JSON
├── migrations/              # Creates the lamaran table
├── test/                    # Health, validation & integration checks
├── scripts/
│   └── migrate-lamaran.js   # One-time cutover from old SQLite data
├── infra/
│   ├── Caddyfile            # Front door routing for :7014
│   ├── docker-compose.stack.yml
│   └── scripts/             # start.sh & stop.sh — the two buttons you need
├── .github/workflows/       # Runs lint + tests on every push
├── Dockerfile               # Recipe to run this inside Docker
├── .env.example             # Clean template for configuration
├── docs.md                  # The long-form documentation
└── .gitignore               # Keeps secrets and personal databases out of Git
```

---

## Golden House Rules

1. Keep Secrets Secret: Never commit `.env` to GitHub. Your Postgres password and tokens stay right on your computer.
2. One Driver at a Time: Don't run `npm run dev` and the Docker stack at the same time — they will argue over port 7012!
3. Keep the Auth Server Running: Every private route asks Central Auth (`:7002`) to verify your ticket. If that server is taking a nap, this one answers `503` instead of guessing — so start auth first.
4. Share the Database, Don't Ship It: Postgres is *not* part of this stack. It reuses the container already running on `:5432`, and `docker-compose.stack.yml` points `DB_HOST` at `host.docker.internal` to reach it.
5. Mind the Other Doors: `start.sh` only ever touches this stack. Ports `443`, `8443`, `9443`, and `10000` belong to other services — hands off.

Enjoy building!
# backend-api-services-jobtracker
