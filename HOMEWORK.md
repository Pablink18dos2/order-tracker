# Homework 4 — DevOps and Observability

Fork of [alexeygrigorev/order-tracker](https://github.com/alexeygrigorev/order-tracker).

> **Honest scope note:** this submission is a minimal one. I did **not** build the
> telemetry stack, the alert or the responder yet. The answers below were derived by
> **reading the source code** (`app/main.py`), not by running the stack. Below the
> answers is the plan I would follow to implement it for real.

## Answers

| # | Answer | Reasoning (from `app/main.py`) |
|---|---|---|
| 1 | `{"status":"ok"}` | `/healthz` runs `SELECT 1` and returns `{"status": "ok"}`. |
| 2 | 200 | `standard-1001` is a seeded order; `order_detail` only does date maths for `express` orders. |
| 3 | 404 | The seed data is `standard-1001`, `express-1002`, `standard-1003`. `standard-1002` does not exist, so `get_order` raises `HTTPException(404)`. |
| 4 | Normal | A 404 is a client error, not a 5xx, so the 5xx alert does not fire. Alert is configured to treat "no 5xx" as 0 (not "No data"). |
| 5 | Not run | Needs the responder service and a headless coding agent. Expected: it reports a test notification with no incident and changes no code. |
| 6 | The express delivery date calculation tried to use a day that does not exist in that month. | `express-1002` is seeded with `created_at` = last day of the previous month (28–31). `order_detail` does `placed_at.replace(day=placed_at.day + 2)` → day 30–33 does not exist → `ValueError` → HTTP 500. Fix: `placed_at + timedelta(days=2)`. |

## What I would do to implement it (plan)

1. **Run it:** `docker compose up --build -d --wait`, then `curl localhost:8000/healthz`.
2. **Instrument (Q2):** add OpenTelemetry (`opentelemetry-sdk`, FastAPI instrumentation)
   for metrics, logs and traces; the request counter carries `http.route` and
   `http.status_code`. Export to console first and read it with `docker compose logs app`.
3. **Pipeline (Q3):** add to `compose.yaml` an OpenTelemetry Collector, Prometheus
   (metrics), Loki (logs), Tempo (traces) and Grafana, with the config committed under
   `observability/`. The app sends OTLP to the Collector, which fans out to the three
   stores. Provision a Grafana dashboard for request count and 5xx errors.
4. **Alert (Q4):** Grafana alert rule on `sum(rate(http_requests_total{status=~"5.."}[5m]))`,
   with endpoint, window and dashboard link in annotations, and "no data → OK".
5. **Responder (Q5):** small FastAPI service in `incident-response/` listening on
   `POST /alerts` (port 8001). It saves the alert, recent logs and traces to disk, then
   launches a coding agent in headless mode (`claude -p "<prompt>"`) with those files.
   Prompt: investigate root cause; if it is a real bug make the smallest fix and run the
   tests; if it is a false positive, explain and change nothing.
6. **Close the loop (Q6):** point a Grafana webhook contact point at the responder,
   call `GET /api/orders/express-1002` until the alert fires, let the agent patch the
   date calculation, restart the app and confirm the request returns 200.
