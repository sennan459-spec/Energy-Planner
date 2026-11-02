# Energy-Planner

> Find the cleanest hours to charge your EV or run your appliances, based on 96-hour carbon intensity forecasts for your grid region.

**Status:** in planning. This README describes the app as designed.

[Live demo](#) · [Design notes](https://claude.ai/artifact/DiFhv4HBqA8wugJF55n3Q8)

---

## What it does

The carbon intensity of electricity (grams of CO₂ per kWh) changes hour by hour as solar, wind, gas and coal plants ramp up and down. Running the same load at a cleaner hour can cut its emissions a lot, at no extra cost.

GridWise lets you:

- **Pick your grid region** (for example CAISO or PJM) and see the next 96 hours of forecast carbon intensity.
- **Ask for the best window** for a task: "charge my EV for 4 hours at 7 kW, done by 7 am".
- **Save alerts** and get an email before your cleanest window starts.

Forecasts come from [CarbonCast](https://github.com/UMass-LIDS/CarbonCast), a neural-network model for multi-day carbon intensity forecasting.

## How it works

```
 Website (Next.js)          Job scheduler (sibling project)
        │                              │
        └──────────────┬───────────────┘
                       ▼
        Shared API (FastAPI, Python)
        /forecast   /best-window   /schedule
                       │
          scheduler core: forecast_greedy
                       │
                       ▼
        Postgres (Supabase)  ◄──  Daily job (GitHub Actions)
        forecasts, users,         pull grid data → run CarbonCast
        saved alerts              → write 96h forecast → send alerts
```

1. Once a day, a GitHub Actions job pulls the latest grid data, runs CarbonCast, and writes a 96-hour forecast per region to Postgres.
2. The FastAPI service reads those forecasts and finds the lowest-emission window for a task, given its length, power draw and deadline.
3. The Next.js site shows the forecast chart and the recommended window, and lets signed-in users save alerts.
4. After each forecast refresh, the daily job emails users whose alert window is coming up.

An EV charge is treated as a "job" with a duration, power and deadline, so the website uses the same scheduling code as the [carbon-aware job scheduler](../scheduler/).

## Tech stack

| Layer | Choice |
| --- | --- |
| Frontend | Next.js, TypeScript, React, hosted on Vercel |
| API | Python, FastAPI, Docker, hosted on Fly.io or Render |
| Database and login | Supabase (Postgres, Auth, row-level security) |
| Forecasting | CarbonCast (TensorFlow/Keras), run daily in GitHub Actions |
| Email | Resend or Postmark |
| Analytics | Plausible or Vercel Analytics |

## Project layout *(planned)*

```
carbon-app/
├── web/          # Next.js frontend
├── api/          # FastAPI service (wraps the shared scheduler core)
├── pipeline/     # daily forecast job: fetch data, run CarbonCast, load Postgres
├── db/           # SQL schema and migrations
└── .github/workflows/
    └── daily-forecast.yml
```

## Getting started *(planned)*

### Prerequisites

- Node.js 20+
- Python 3.11+
- Docker
- A free Supabase project

### Environment variables

Copy `.env.example` to `.env` in `web/`, `api/` and `pipeline/` and fill in:

| Variable | Used by | What it is |
| --- | --- | --- |
| `SUPABASE_URL` | all | Your Supabase project URL |
| `SUPABASE_ANON_KEY` | web | Public key for the browser |
| `SUPABASE_SERVICE_ROLE_KEY` | api, pipeline | Server-side key (never ship to the browser) |
| `EIA_API_KEY` | pipeline | Free key from the [EIA Open Data API](https://www.eia.gov/opendata/) |
| `EMAIL_API_KEY` | pipeline | Resend or Postmark key for alert emails |
| `NEXT_PUBLIC_API_URL` | web | URL of the FastAPI service |

### Run locally

```bash
# API
cd api
pip install -r requirements.txt
uvicorn app.main:app --reload        # http://localhost:8000/docs

# Frontend
cd web
npm install
npm run dev                          # http://localhost:3000

# Load one day of forecasts
cd pipeline
pip install -r requirements.txt
python run_daily.py --region CISO
```

### Tests

```bash
cd api && pytest
cd web && npm test
```

## API *(planned)*

| Method | Path | Returns |
| --- | --- | --- |
| `GET` | `/forecast/{region}` | 96 hourly forecast values (gCO₂/kWh) for a region |
| `GET` | `/best-window?region=CISO&duration_h=4&power_kw=7&deadline=...` | Start and end of the cleanest window, and the emissions saved versus starting now |
| `POST` | `/schedule` | Start times for a batch of jobs (used by the job scheduler) |

Interactive docs are served at `/docs` by FastAPI.

## Roadmap

- [ ] Daily forecast job for one region (CAISO)
- [ ] `/forecast` and `/best-window` endpoints
- [ ] Region page with forecast chart and best-window picker
- [ ] Login and saved alerts
- [ ] Alert emails
- [ ] More regions (PJM, ERCOT, ...)
- [ ] Analytics and public launch

## Data and credits

- Forecast model: Maji, Shenoy and Sitaraman, "CarbonCast: Multi-Day Forecasting of Grid Carbon Intensity", BuildSys '22. Code: [UMass-LIDS/CarbonCast](https://github.com/UMass-LIDS/CarbonCast).
- US grid data: [U.S. Energy Information Administration (EIA)](https://www.eia.gov/opendata/).
- CarbonCast's license still needs checking before its forecasts are shown on a public site.

## License

TBD.
