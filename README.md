# BoxTrace – Cloud-Based Container Shipment Tracking System

A full-stack, container-ready web application that lets shipping operators register containers, create shipments,
ingest live position and temperature data from devices, and lets customers track their cargo on a map with
ETA prediction and automatic alerts.

## Features
- **Public tracking page**: search by container number (ISO 6346) or shipment reference; map, route progress, predicted arrival, event history.
- **Accounts and roles**: customers see only their own shipments; administrators manage containers, shipments and events.
- **ISO 6346 validation**: container numbers are checked with the standard check-digit algorithm.
- **Device ingestion API**: telematics units or AIS feeds POST events with an `X-API-Key`.
- **ETA prediction and alerts**: delay (24 h warning, 72 h critical), reefer temperature excursion, and stale-position alerts.
- **Dashboard**: counters, open alerts, and a shipment table.
- **Cloud-ready**: 12-factor config, health check, Dockerfile, docker-compose, Procfile, AWS ECS Fargate guide, CI workflow.
- **Simulator**: `scripts/simulator.py` moves a container along a real great-circle route to demo live tracking.

## Tech stack
Python 3.12 · Flask · SQLite (swap for PostgreSQL/RDS in `app/db.py`) · Gunicorn · Vanilla JS + Leaflet · Docker.

## Project layout
```
app/            Flask application (api.py, services.py, iso6346.py, seed.py, schema.sql, static/ UI)
scripts/        simulator.py (device simulator)
tests/          unittest suite (15 tests)
deploy/aws/     ECS Fargate task definition and guide
docs/           API.md, PROJECT_REPORT.md
Dockerfile, docker-compose.yml, Procfile, wsgi.py, requirements.txt, .env.example
```

## Run locally
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
export SEED_DEMO=true                                   # Windows PowerShell: $env:SEED_DEMO="true"
python -m flask --app wsgi run --port 8000
```
Open http://localhost:8000. Demo logins:

| Role | Email | Password |
|---|---|---|
| Admin | admin@boxtrace.test | admin12345 |
| Customer | customer@boxtrace.test | customer12345 |

Demo containers to search: `BXTU2045182` (on time), `BXTU3529068` (delayed), `BXTU7781205` (reefer temperature alert),
`BXTU6104473` (arrived), `BXTU9932152` (booked), `BXTU1288737` (transshipment).

## Run with Docker
```bash
cp .env.example .env        # edit secrets; keep SEED_DEMO=true for the demo data
docker compose up --build
```

## Live-tracking demo
With the server running, in a second terminal:
```bash
python scripts/simulator.py --container BXTU9932152 --from KRPUS --to USLGB --steps 20 --interval 1
```
Open the tracking page for `BXTU9932152` and reload to watch it move from Busan to Long Beach.

## Tests
```bash
python -m unittest discover -s tests -t . -v
```

## Configuration (environment variables)
| Variable | Default | Meaning |
|---|---|---|
| `SECRET_KEY` | dev value | Signs login tokens. **Set a long random value in production.** |
| `DEVICE_API_KEY` | `dev-device-key` | Key that devices send in `X-API-Key` |
| `DATABASE_PATH` | `./data/tracking.db` | SQLite file location |
| `TOKEN_MAX_AGE` | `86400` | Login token lifetime in seconds |
| `AVG_SPEED_KMH` | `24` | Vessel speed used for ETA prediction |
| `CORS_ORIGIN` | empty | Allowed origin if the UI is hosted separately |
| `SEED_DEMO` | `false` | Load demo data when the database is empty |

## Cloud deployment
See `deploy/aws/README.md` (ECS Fargate + ALB + EFS + Secrets Manager). The same image runs on Azure Container Apps,
Google Cloud Run, Render, Railway or Heroku.

## Known limitations
- ETA uses great-circle distance at a constant speed, so it ignores real shipping lanes, canals and weather.
  Progress and ETA are therefore estimates; feed real AIS data or a routing service to improve them.
- One container per shipment (no multi-container bookings or bills of lading).
- SQLite suits a single running instance. Use PostgreSQL (RDS) before scaling to several instances.
- Login tokens live in `localStorage`; for a public production site consider HTTP-only cookies and rate limiting on `/auth/*`.
- Demo credentials and the default keys must be changed before any real deployment.
