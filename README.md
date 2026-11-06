# EMA Green Seal Portal

A web portal where film and TV productions apply for the Environmental Media Association's **Green Seal** and **Gold Seal**. Productions answer the PEACH sustainability checklist and report their footprint. Reviewers verify the answers and decide each application. Staff manage the program, see dashboards and reports, and issue certificates.

- **Live site:** https://jmsouch2896.github.io/greenseal/
- **Backend:** Supabase project `pdgfeniatfaktiwdabrh` (Postgres, Auth, Storage, Realtime, Edge Functions)
- **Hosting:** GitHub Pages, served from the `main` branch root

## How it's built

| Layer | What | Where |
|---|---|---|
| Front end | One self-contained page: HTML, CSS and vanilla JavaScript, with no build step | [`index.html`](index.html) |
| Data and rules | Postgres tables protected by row-level security (RLS); triggers enforce workflow rules and write the activity log | Supabase (see [docs/database.md](docs/database.md)) |
| Server code | Four Deno edge functions: account admin, email, certificates, geocoding | Supabase (see [docs/edge-functions.md](docs/edge-functions.md)) |
| Files | `evidence` bucket (private) for uploaded proof, `branding` bucket (public) for logos and signatures | Supabase Storage |

The browser talks to Supabase directly with the **publishable** key. The key is meant to be public. Every request runs as the signed-in user, so the database's RLS policies decide what each person can read and change. Anything that needs elevated rights runs in an edge function, which checks the caller first.

## Documentation

| Doc | Read it for |
|---|---|
| [docs/architecture.md](docs/architecture.md) | The big picture: roles, the security model, the application lifecycle, external services |
| [docs/frontend.md](docs/frontend.md) | How `index.html` is organized: state, routing, rendering, charts, the dashboard, theming |
| [docs/database.md](docs/database.md) | Tables, security helpers, triggers, RPCs, settings keys, storage, migration history |
| [docs/edge-functions.md](docs/edge-functions.md) | Each edge function's inputs, outputs, permissions and secrets |
| [docs/development.md](docs/development.md) | Running locally, making changes safely, testing, deploying |

## Quick start

```bash
git clone https://github.com/Jmsouch2896/greenseal.git
cd greenseal
python3 -m http.server 8000      # any static server works
# open http://localhost:8000
```

The page connects to the live Supabase project, so signing in uses real accounts and real data. Read [docs/development.md](docs/development.md) before changing anything.

## Deploying

Push to `main`. GitHub Pages republishes in about a minute. Database changes and edge functions are deployed separately to Supabase; see [docs/development.md](docs/development.md#deploying).

## Roles at a glance

| Role | Can |
|---|---|
| **submitter** (default) | Create applications, answer the PEACH checklist, report the footprint, upload evidence, invite department delegates, submit |
| **reviewer** | Everything on submitted applications: verify points, flag actions for changes, comment, change status, see the dashboard, Analysis and report |
| **staff** | Everything, plus questions and form versions, users, contacts, companies, the portal look and certificate design |

A **delegate** is a submitter invited to answer specific departments on someone else's draft.
