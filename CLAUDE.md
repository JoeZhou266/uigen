# CLAUDE.md — okta-perf

Performance-test tool for RESTful APIs protected by **Okta OAuth2 (client credentials flow)**.
It drives a configured request rate (requests per minute) with a bounded concurrency level,
records every request, and produces console, JSON, CSV and HTML reports.

Read this file fully before making changes. When a rule here conflicts with a habit, follow this file.

---

## 1. Tech stack

| Concern            | Choice                                   |
|--------------------|------------------------------------------|
| Python             | 3.12 (managed with `uv`)                 |
| HTTP client        | `httpx` (async, HTTP/1.1; HTTP/2 optional) |
| Concurrency        | `asyncio` only — no threads, no multiprocessing in v1 |
| Config             | `pydantic` v2 + `pydantic-settings`, YAML scenario files (`pyyaml`) |
| CLI                | `typer` + `rich` (live progress, summary tables) |
| Stats              | `numpy` for percentiles                  |
| Reports            | `jinja2` HTML template, `matplotlib` charts embedded as inline SVG |
| Tests              | `pytest`, `pytest-asyncio`, `respx` (httpx mocking) |
| Quality            | `ruff` (lint + format), `mypy --strict`  |

Do not add new runtime dependencies without asking first.

---

## 2. Commands

```bash
uv sync                                   # install deps
uv run okta-perf run scenarios/sample.yaml           # run a test
uv run okta-perf run scenarios/sample.yaml --dry-run # validate config + fetch token only
uv run okta-perf report results/<run-id>/raw.jsonl   # rebuild reports from raw data
uv run pytest -q                          # tests
uv run pytest --cov=okta_perf --cov-report=term-missing
uv run ruff check . && uv run ruff format --check .
uv run mypy src
```

Before declaring any task done: `ruff check`, `ruff format`, `mypy src`, `pytest` must all pass.

---

## 3. Project layout

```
okta-perf/
├── CLAUDE.md
├── pyproject.toml
├── .env.example                 # OKTA_* placeholders, never real values
├── scenarios/
│   └── sample.yaml
├── src/okta_perf/
│   ├── __init__.py
│   ├── cli.py                   # typer app: run, report, validate
│   ├── config.py                # Pydantic models: Settings (env) + Scenario (YAML)
│   ├── auth/
│   │   └── okta.py              # TokenProvider: client-credentials fetch, cache, refresh
│   ├── engine/
│   │   ├── scheduler.py         # rate scheduler (RPM -> arrival times, ramp-up)
│   │   ├── runner.py            # orchestrates workers, semaphore, shutdown
│   │   └── request_builder.py   # endpoint selection by weight, templating
│   ├── metrics/
│   │   ├── models.py            # RequestResult dataclass
│   │   ├── collector.py         # async queue -> JSONL writer + in-memory aggregates
│   │   └── stats.py             # percentiles, throughput, error rates, per-minute buckets
│   └── reporting/
│       ├── console.py
│       ├── json_report.py
│       ├── csv_report.py
│       ├── html_report.py
│       └── templates/report.html.j2
├── results/                     # git-ignored; one folder per run-id
└── tests/
    ├── conftest.py              # fake Okta + fake target via respx, fake clock
    ├── unit/
    └── integration/
```

---

## 4. Okta client credentials flow (src/okta_perf/auth/okta.py)

**Token request**

- `POST {OKTA_ISSUER}/v1/token` where issuer is e.g. `https://{domain}/oauth2/default`
  (custom authorization server — the org server `/oauth2/v1/token` does not support custom scopes).
- Form body: `grant_type=client_credentials&scope=<space-separated scopes>`
- Client auth: `client_secret_basic` (HTTP Basic header) by default.
  Support `private_key_jwt` as a config option (sign a JWT assertion with the private key; `PyJWT[crypto]`
  is acceptable **only** when that option is implemented — ask first).
- Header `Accept: application/json`, `Content-Type: application/x-www-form-urlencoded`.

**TokenProvider rules**

1. One shared provider per run. Workers call `await provider.get_token()`.
2. Cache the token in memory; refresh when `now >= expires_at - refresh_skew` (default skew 60 s).
3. Single-flight refresh: guard with an `asyncio.Lock` so N concurrent workers trigger **one** Okta call.
4. On a `401` from the target API, invalidate the token, refresh once, retry the request once. Never loop.
5. Retry token fetch on network errors / 5xx / 429 with exponential backoff (max 3 attempts, honor `Retry-After`).
6. Token fetch time is recorded separately (`auth_latency_ms`) and **never** counted in API latency stats.
7. Okta rate-limits `/token` — never fetch a token per request. A test should assert the fetch count.
8. `--dry-run` fetches one token, prints its `expires_in` and granted scopes (decoded, not verified), then exits.

**Secrets**

- `OKTA_CLIENT_ID`, `OKTA_CLIENT_SECRET` (or `OKTA_PRIVATE_KEY_PATH`), `OKTA_ISSUER`, `OKTA_SCOPES` come
  from environment / `.env` only — never from YAML, never from CLI args.
- Use `pydantic.SecretStr`. Never log, print, or write tokens or secrets to reports. Redact the
  `Authorization` header anywhere a request is serialized.

---

## 5. Load model (src/okta_perf/engine)

The tool uses an **open (arrival-rate) model**: the scheduler emits requests at a target rate regardless of
how fast responses come back. Concurrency is a **cap**, not the driver.

**Scenario parameters**

| Field                  | Meaning                                                      |
|------------------------|--------------------------------------------------------------|
| `rate.requests_per_minute` | Target arrival rate (e.g. 600 = 10 req/s)                |
| `rate.max_concurrency` | Max in-flight requests (`asyncio.Semaphore`)                  |
| `rate.duration`        | Test length, e.g. `5m`, `90s`                                 |
| `rate.ramp_up`         | Linear ramp from 0 to target, e.g. `30s` (optional)           |
| `rate.arrival`         | `constant` (evenly spaced, default) or `poisson`              |
| `rate.max_queue_wait`  | If a slot is not free within this time, record `dropped` (default `5s`) |

**Scheduler rules**

1. Compute each request's **intended send time** from the rate; sleep until it with `asyncio.sleep` against
   `time.monotonic()`. Do not accumulate drift (compute from start time, not from the previous sleep).
2. Record both `intended_at` and `sent_at`. Report latency both as **service time** (`sent → response`)
   and **response time** (`intended → response`) to avoid coordinated omission.
3. If the semaphore is saturated, the request waits up to `max_queue_wait`, then is counted as `dropped`.
   Report `achieved RPM` vs `target RPM` so saturation is visible.
4. Graceful shutdown on duration end or Ctrl+C: stop scheduling, wait for in-flight requests (timeout
   `drain_timeout`, default 30 s), then flush metrics and write reports.
5. Use one shared `httpx.AsyncClient` with `httpx.Limits(max_connections=max_concurrency)` and explicit
   timeouts (connect / read / write / pool) from config.

**Sample scenario (scenarios/sample.yaml)**

```yaml
name: orders-api-baseline
target:
  base_url: https://api.example.com
  timeout_seconds: 10
  verify_tls: true
rate:
  requests_per_minute: 1200
  max_concurrency: 50
  duration: 5m
  ramp_up: 30s
  arrival: constant
endpoints:
  - name: list-orders
    method: GET
    path: /v1/orders?page=1&size=20
    weight: 70
    expect_status: [200]
  - name: create-order
    method: POST
    path: /v1/orders
    weight: 30
    headers: { Content-Type: application/json }
    body_file: payloads/create-order.json     # supports {{uuid}}, {{now_iso}}, {{seq}}
    expect_status: [201]
thresholds:                                   # SLA gates -> non-zero exit code on breach
  p95_ms: 500
  p99_ms: 1200
  error_rate_pct: 1.0
  min_achieved_rpm_pct: 95
report:
  formats: [console, json, csv, html]
  output_dir: results
```

---

## 6. Metrics & reports

**Per-request record (`RequestResult`)** — written as one JSON line to `results/<run-id>/raw.jsonl`:
`seq, endpoint, method, status, outcome (ok|http_error|timeout|conn_error|dropped|auth_error),
intended_at, sent_at, finished_at, service_ms, response_ms, bytes_in, error (short message)`.

The collector consumes an `asyncio.Queue`; workers never do file I/O directly.

**Aggregates** (overall and per endpoint)

- Total / ok / failed / dropped counts, error rate %, status-code distribution, error types
- Target RPM vs achieved RPM, peak in-flight concurrency
- Latency: min, mean, p50, p90, p95, p99, max — for both service time and response time
- Per-minute timeline: RPM, error rate, p95
- Token stats: fetch count, avg auth latency, refresh count, 401-retry count

**Outputs in `results/<run-id>/`** (`run-id` = `YYYYMMDD-HHMMSS-<scenario-name>`)

- `raw.jsonl` — source of truth; every report can be regenerated from it
- `summary.json` — aggregates + scenario (secrets excluded) + threshold verdicts
- `summary.csv` — one row per endpoint + one `ALL` row
- `report.html` — single self-contained file: header (scenario, times, verdict), summary table,
  latency percentile chart, per-minute RPM/p95/error chart, status distribution, threshold table.
  Charts are inline SVG; no CDN or network assets.
- Console: `rich` live progress during the run; summary table + PASS/FAIL at the end.

**Exit codes**: `0` pass, `1` threshold breach, `2` config/validation error, `3` auth failure (cannot get token).

---

## 7. Coding conventions

- Full type hints; `mypy --strict` clean. No `Any` without a comment explaining why.
- Pydantic models for all config and external data; dataclasses (`slots=True`) for hot-path records.
- Async all the way down in the engine; never call blocking I/O inside the event loop
  (report generation runs after the loop finishes).
- Use `time.monotonic()` for durations, `datetime.now(UTC)` for timestamps in output.
- Small, pure functions in `stats.py` so they are unit-testable without I/O.
- Logging via standard `logging`, configured in `cli.py`; default level INFO, `--verbose` for DEBUG.
  No per-request logs at INFO level.
- Docstrings on public functions (Google style). Keep modules under ~300 lines.

---

## 8. Testing strategy

- **Never** hit real Okta or a real target in the test suite. Use `respx` to mock both.
- `conftest.py` provides: `fake_okta` (issues tokens with configurable `expires_in`, counts calls),
  `fake_target` (configurable latency, status mix, 401 injection), and a short scenario factory.
- Required tests:
  - token cached; single-flight under 100 concurrent callers → exactly 1 Okta call
  - token refreshed before expiry; 401 triggers one refresh + one retry
  - Okta 429/5xx retry with backoff; auth failure → exit code 3
  - scheduler produces the right number of requests ±2% for a short run; ramp-up shape correct
  - concurrency never exceeds `max_concurrency`; saturation produces `dropped`
  - percentile math matches `numpy` reference on known data
  - reports: JSON schema, CSV columns, HTML renders and contains no secrets / no `Bearer ` strings
  - threshold evaluation and exit codes
- Keep the full suite under ~30 s: use short durations (1–3 s) and high RPM in tests.

---

## 9. Safety rules

- Refuse to run against a host listed in `PROTECTED_HOSTS` (env, comma-separated) unless
  `--i-know-this-is-prod` is passed. Print target host, RPM and duration and require confirmation
  (`--yes` skips it for CI).
- Default `max_concurrency` cap of 500 and `requests_per_minute` cap of 60 000 in config validation;
  higher values require an explicit override flag.
- `.env`, `results/`, and `*.pem` are git-ignored. Only `.env.example` is committed.

---

## 10. Build order (for a fresh implementation)

1. `pyproject.toml`, tooling config, `config.py` with scenario validation + `validate` command
2. `auth/okta.py` + tests, `--dry-run`
3. `metrics/models.py`, `collector.py`, `stats.py` + tests
4. `engine/scheduler.py`, `request_builder.py`, `runner.py` + tests
5. Console + JSON + CSV reports, thresholds, exit codes
6. HTML report
7. `report` command (regenerate from `raw.jsonl`), README

Work one step at a time; each step ends with tests passing and a short summary of what changed.

---

## 11. Out of scope (v1)

Distributed / multi-machine load generation, gRPC or WebSocket targets, authorization-code or PKCE
flows, real-time dashboards (Grafana/Prometheus export may come in v2 — keep `stats.py` export-friendly).
