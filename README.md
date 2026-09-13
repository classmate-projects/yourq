<div align="center">

# YourQ

**A simple, no-frills token queue system for service counters.**

[![Node.js](https://img.shields.io/badge/Node.js-%3E%3D18-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Express](https://img.shields.io/badge/Express-4.x-000000?logo=express&logoColor=white)](https://expressjs.com/)
[![EJS](https://img.shields.io/badge/View%20Engine-EJS-a91e50)](https://ejs.co/)
[![Supabase](https://img.shields.io/badge/Storage-Supabase%20%2F%20Postgres-3ECF8E?logo=supabase&logoColor=white)](https://supabase.com/)
[![Deploy](https://img.shields.io/badge/Deploy-Netlify%20%7C%20Render-informational)](#hosting-it-online)

</div>

---

## The idea

Walk into a pharmacy, clinic, or government office and you'll usually find one
of two things: a paper ticket dispenser, or nothing at all — just people
hovering near the counter hoping to be noticed next. **YourQ** is a lightweight
digital stand-in for the ticket dispenser: customers tap a service to get a
number, staff at each desk call the next number in order, and a shared display
board announces who's up — with a chime. No app to install, no accounts to
create; it just runs in a browser on whatever's already at the counter.

It's built to be **boringly reliable**: a handful of Express routes, server-rendered
pages so there's no loading flash, and a storage layer that gracefully falls
back to a local JSON file when there's no database configured at all.

## Table of contents

- [Pages](#pages)
- [Features](#features)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Setup](#setup)
- [Local development via yourq-api (optional)](#local-development-via-yourq-api-optional)
- [Hosting it online](#hosting-it-online)
- [How it works](#how-it-works)
- [Notes](#notes)
- [Related project](#related-project)

## Pages

| Page | Route | Who it's for |
|---|---|---|
| **Home** | `/` | Everyone — a landing hub linking to the four pages below |
| **Customer** | `/customer` | Customers — tap a service, then **OK** to take a token |
| **Now Serving board** | `/display` | The waiting area — shows every desk and the token it's calling, live, with a chime |
| **Desk control** | `/desk` | Staff — **Call Next** (FIFO) and **Recall** the token they're serving |
| **Admin** | `/admin` | Managers — password-protected: manage services/desks, download reports |

All pages are rendered server-side by Express/EJS (`views/`), so the initial
state is baked into the HTML with no loading flash. Client JS (`public/js/`)
handles interactions and refreshes the live numbers — instantly on a
persistent server, or via a ~2s poll on Netlify. Adding, deleting, or changing
a service (including its desk count) reloads the affected pages so the
server-rendered markup stays the source of truth.

## Features

**Customer-facing**
- Grid of active services; tap to pick one, **OK** to take a token
- Per-service, per-day token numbering that resets automatically at local midnight

**Now Serving board**
- Live view of every desk's current token, per service
- Two-tone chime on a new call, a distinct triple-beep on a recall
- Theme-aware (follows the device's light/dark setting)

**Desk control**
- FIFO **Call Next** — can't call past the last token taken
- **Recall** — re-announces the desk's current token without advancing the queue
- Works across any number of desks per service

**Admin**
- Add/remove services and set each service's desk count
- Daily CSV report of customers served per service, for any date
- Subscription status (Free/Pro) with a renew reminder near expiry

**Under the hood**
- Server-rendered pages (no client-side framework, no build step)
- Storage auto-detects its backend: local dev API → Supabase → local JSON file
- Auto-downgrades a Pro subscription to Free when it expires — no manual step
- Distinct, stable colors per service across every page

## Tech stack

| Layer | Choice |
|---|---|
| Runtime | Node.js 18+ |
| Server | Express |
| Views | EJS (server-rendered) |
| Client JS | Vanilla JS (`public/js/`), no framework |
| Storage (production) | Supabase (hosted Postgres) |
| Storage (local dev, optional) | [yourq-api](#related-project) → local Postgres |
| Storage (no setup) | Local JSON file |
| Hosting | Netlify (serverless) or Render/Railway/Fly.io (persistent) |

## Project structure

```
yourq/
├── app.js                # Express app: routes, pages, API
├── server.js             # Local entry point (npm start)
├── netlify/functions/     # Same app, wrapped for Netlify's serverless runtime
├── views/                # EJS templates (home, index, display, desk, admin)
├── public/                # Static assets: css/, js/, favicon
├── src/
│   ├── db.js              # Queue/service business logic
│   ├── storage.js         # Persistence backend (dev API / Supabase / file)
│   ├── auth.js             # Admin password auth
│   ├── subscription.js     # Free/Pro plan limits & expiry evaluation
│   └── util.js
├── supabase/schema.sql    # Production database schema
├── netlify.toml           # Netlify build/deploy config
└── render.yaml            # Render Blueprint config
```

## Setup

1. Install [Node.js](https://nodejs.org/) 18 or later.
2. Install dependencies:
   ```
   npm install
   ```
3. Copy `.env.example` to `.env` and set your own admin password:
   ```
   copy .env.example .env
   ```
   Then edit `.env`:
   ```
   PORT=3000
   ADMIN_PASSWORD=your-own-password
   ```
4. Start the server:
   ```
   npm start
   ```
   (`npm run dev` runs the same thing but auto-restarts when you edit a file.)
5. Open the pages above in a browser. Since everything runs on your local
   machine, other devices on the same WiFi network can reach it too using your
   computer's local IP instead of `localhost` (e.g. `http://192.168.1.23:3000/`).

## Local development via yourq-api (optional)

By default (`.env` with no `DEV_API_URL`/`SUPABASE_URL`), the app stores data in
a local JSON file — no setup needed. If you'd rather develop against a real
Postgres database locally, route the app through the sibling **[yourq-api](#related-project)**
(.NET) project instead of Supabase:

1. Create a local Postgres database, e.g. `createdb yourq`.
2. In `yourq-api/appsettings.Development.json`, point `ConnectionStrings:Postgres`
   at it.
3. Run the API: `dotnet run --project ../yourq-api` (it creates the tables on
   startup — see `yourq-api/Database/schema.sql`). It listens on
   `http://localhost:5103` by default.
4. In this project's `.env`, set `DEV_API_URL=http://localhost:5103` and leave
   `SUPABASE_URL`/`SUPABASE_KEY` blank.
5. `npm start` as usual — `db.js`/`storage.js` are unchanged; only the storage
   backend they talk to differs.

This path is **dev-only**. Production keeps talking to Supabase directly (see
below) — `yourq-api` is never deployed alongside it.

## Hosting it online

The app can be deployed two ways. **The code auto-detects where it runs** — no
edits needed.

### Option A — Netlify (serverless) + Supabase

`netlify.toml` runs the whole app as one serverless function and serves the
CSS/JS statically. Data persists to **Supabase** (a hosted Postgres database),
stored as one JSON document in a `queue_state` table. Live updates use a
~2-second poll instead of WebSockets (serverless can't hold sockets open).

1. **Create the database.** In [supabase.com](https://supabase.com), create a
   project, open **SQL Editor**, paste the contents of
   [`supabase/schema.sql`](supabase/schema.sql), and run it.
2. **Get your keys.** In Supabase → **Project Settings → API**, copy the
   **Project URL** and the **secret** API key (starts with `sb_secret_…`, not
   the publishable one — the server needs full access and the table is locked
   with RLS).
3. Push this repo to GitHub.
4. On [netlify.com](https://netlify.com): **Add new site → Import an existing
   project**, pick the repo. Netlify reads `netlify.toml`, so leave the build
   settings as detected (build `npm install`, publish `public`, functions
   `netlify/functions`). If the site was created earlier with different
   settings, clear any "Publish directory" / "Build command" overrides in
   **Site configuration → Build & deploy**.
5. In **Site configuration → Environment variables**, add:
   - `ADMIN_PASSWORD` — your admin password
   - `SUPABASE_URL` — the Project URL
   - `SUPABASE_KEY` — the secret key
6. Deploy, then open the `https://…netlify.app/` URL. Add a service in Admin,
   then check **Supabase → Table Editor → queue_state** to confirm it saved.

If `SUPABASE_URL`/`SUPABASE_KEY` are not set, the app falls back to a local
JSON file (fine for `npm start`, but not persistent on serverless).

### Option B — Render / Railway / Fly.io (persistent server)

These run the app as a normal long-lived server (`npm start`), which keeps
instant updates instead of polling. Set `SUPABASE_URL`/`SUPABASE_KEY` for
persistent storage (recommended — free-tier disks are ephemeral); otherwise it
uses a local file.

1. Push to GitHub.
2. On [render.com](https://render.com): **New → Blueprint** (reads
   `render.yaml`), or **New → Web Service** with build `npm install` / start
   `npm start`.
3. Set `ADMIN_PASSWORD` in the service's environment.
4. Open the `https://…onrender.com/` URL.

On free tiers the disk is *ephemeral*, so data resets on restart — attach a
persistent disk/volume and point `QUEUE_DB_PATH` at it (see `render.yaml`).

## How it works

- **Two counters per service, per day.** *Issued* counts tokens customers have
  taken; *called* is how far the desks have served. Call Next advances *called*
  (up to *issued*) and assigns that token to the calling desk. Both reset
  automatically at **local midnight**, so the first token each day is `#1`. The
  timezone defaults to `Asia/Dubai` (UTC+4) and is set via the `QUEUE_TIMEZONE`
  env var — important on serverless hosts, which run in UTC.
- **Multiple desks.** Each service has a number of desks set in admin. Every desk
  serves from the same FIFO queue but tracks the token it personally called, so
  the board can show "Token 15 → Desk 2".
- **Data** is stored in `data/db.json`, created automatically on first run. Set
  `QUEUE_DB_PATH` to store it elsewhere. Back it up to keep history beyond the
  daily report.
- **Deleting a service** removes it from the customer/desk/board pages but keeps
  its history so past daily reports stay accurate.
- **The daily report** is a CSV download (`Service,Customers Served`) for a date
  you pick, defaulting to today. "Served" counts tokens taken (customers arrived).
- **Subscription (Free / Pro).** Stored in the Supabase `subscription` table
  (columns `plan` and `expiry_date`). At the daily boundary — the same local
  midnight as the token reset — the app re-evaluates it: when a Pro plan's
  `expiry_date` is reached or passed, it **auto-downgrades to Free** (no manual
  step). **Upgrading is manual:** the admin sets `plan = 'pro'` and a future
  `expiry_date` directly on that row in Supabase. Visibility is role-specific:
  the **Admin** page shows the plan, days remaining, and a renew alert on each
  login; the **Desk** page shows a single dismissible reminder once per day when
  expiry is within 7 days; the **customer (kiosk)** and **display board** show
  nothing about the subscription.
- **Colors.** Each service gets a distinct color (assigned in creation order)
  used consistently across the customer grid, board, desk page, and admin list.
  Pages are theme-aware and follow the device's light/dark setting.

## Notes

- Designed to run on one machine on a trusted local network (e.g. a shop or
  clinic counter). The admin login is a single shared password; the customer and
  desk pages are open on the local network. This is enough to keep casual
  visitors from editing services — not a substitute for real access control if
  the app is exposed to the internet.

## Related project

- **[yourq-api](https://github.com/classmate-projects/yourq-api)** — a small
  .NET API used only in local development, so this app can develop against a
  real local Postgres database instead of production Supabase. It lives in its
  own, independent Git repository — it is not a subfolder of this one and is
  never deployed alongside production.
