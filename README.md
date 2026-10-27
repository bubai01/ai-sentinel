# AI Sentinel — Insider Fraud Early Warning + Database Honeypot (Layer 2)

One file (`app.py`) = backend + web UI. Local development uses two SQLite databases in `./data/`:

| Database | Purpose |
|---|---|
| `production.db` | Real data: customers, transactions, employees, alerts, cases, users, settings, audit log |
| `honeypot.db` | Decoy tables (swift_beneficiaries_2019, ceo_salary_ledger, kyc_master_backup, vip_accounts_archive, canary_customers), fake shadow copies of customers/transactions, the central log and honeypot sessions |

## Run
    pip install -r requirements.txt
    python app.py            # or run.bat (Windows) / run.sh (Mac, Linux)
Open http://localhost:8000 — login `admin` / `sentinel123`, registration key `SENTINEL-2026`.

## Deploy on Render with Neon

The Render Blueprint in `render.yaml` builds the included Dockerfile, creates a web service, and generates unique `ADMIN_PASSWORD` and `REG_KEY` secrets. The app creates separate `production` and `honeypot` schemas in Neon when `DATABASE_URL` is set; without it, local development continues to use SQLite.

1. Push this repository to GitHub.
2. In Render, choose **New → Blueprint**, connect the GitHub repository, and apply the `render.yaml` blueprint.
3. In Neon, open the project and copy its **direct (unpooled) PostgreSQL connection string**. In the Render service's environment settings, set `DATABASE_URL` to that value and save. Keep the connection string private.
4. Wait for the Render deploy to finish and check `/healthz`. The health check verifies both database schemas.
5. In Render's environment settings, reveal `ADMIN_PASSWORD` and use it to sign in as `admin`. Keep both generated secrets private.

Render's free web service may sleep when idle. Neon is the persistent cloud database; no Render disk is required.

## How Layer 2 works
1. **Gateway** (`POST /api/db/query`) routes each query: normal tables -> production DB; decoy tables -> honeypot DB.
2. **Central log**: every query is written to the honeypot database (SQLite locally, Neon in deployment).
3. **AI/ML risk scoring**: decoy hits are scored by an explainable model; activity events by an Isolation Forest.
4. **Alert + case**: each decoy hit raises an alert; at the threshold it escalates to a case.
5. **Session isolation**: at/above the threshold the session is isolated, and from then on all of that employee's
   queries are served from the honeypot DB (fake data) and recorded. Terminate = queries refused (403).

## Deploy
Docker: `docker compose up -d --build` (data in a volume). Any Python host: start command `python app.py`,
keep `DATA_DIR` on a persistent disk. Set `ADMIN_PASSWORD` and `REG_KEY`, and serve over HTTPS.
