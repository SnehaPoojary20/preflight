# 🛫 Preflight

**Know before you `ALTER`.**

> Preflight predicts what a database migration will actually do before it runs in production — how long it locks, and which lines of your application code will break.

[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-async-009688)](https://fastapi.tiangolo.com/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Tests](https://img.shields.io/badge/tests-pytest-yellow)](#testing)

---

## The Gap This Fills

- **gh-ost / pt-online-schema-change / Vitess** solve *how to execute* a migration without long locks — they know nothing about your application.
- **Bytebase / Atlas / Flyway / Liquibase** solve *review and governance* — they reason about the schema, not the code that depends on it.

Nobody closes the loop between "this `ALTER` is about to run" and "here is the function that will throw when it does." Preflight does both halves in one pipeline.

## How It Works

1. Preflight spins up a shadow database seeded with realistic data (sampled from a snapshot, or synthesized to match production statistics).
2. It runs your migration against the shadow DB and measures **real** lock duration — not a static heuristic.
3. In parallel, it statically traces your application's ORM/SQL call sites and builds a map of every function that touches the affected columns.
4. It merges both signals into one report: lock impact + exact broken call sites (file, line, reason).

```
Migration PR → Shadow DB Orchestrator → Run migration → Lock & Timing Collector
                                              │
Application source → Static Code Tracer → Column/Table Usage Map
                                              │
                                      Risk Aggregator
                                              │
                          PR comment / CLI output / dashboard
```

Full architecture and algorithm notes: [`docs/architecture.md`](docs/architecture.md) · [`docs/algorithm-notes.md`](docs/algorithm-notes.md)

## Example Output

```
⚠ ALTER TABLE orders DROP COLUMN legacy_status
  Lock: ACCESS EXCLUSIVE, ~38s under production-like load (240k rows)
  Blocks writes: yes

  Broken call sites:
  - orders/service.py:142         SELECT legacy_status  (will raise UndefinedColumn)
  - billing/webhook_handler.py:88 ORM access order.legacy_status
```

## Features

- 🔬 Real, measured lock duration and simulated replication lag — not a heuristic warning
- 🔍 Static tracing of ORM/SQL call sites down to file + line number
- 🧩 Merges execution risk and application risk into a single, ranked report
- 🤖 GitHub Action + CLI — runs in CI, no dashboard required to get value
- 📊 Explicit "unresolved" flagging for anything the static tracer can't confidently resolve
- 📈 Prometheus metrics for shadow-run duration and queue depth

## Tech Stack

**Backend:** FastAPI · Python `ast` · sqlglot
**Orchestration:** Docker SDK for Python (ephemeral shadow databases)
**Queue/Workers:** Redis Streams · arq
**Storage:** PostgreSQL (reports, run history)
**Testing:** pytest · Hypothesis (property-based) · Locust (load)
**Infra:** Docker Compose · GitHub Actions

## Getting Started

```bash
git clone https://github.com/<your-username>/preflight.git
cd preflight
cp .env.example .env
docker-compose up --build
```

Run a check locally via the CLI:

```bash
pip install -e .
preflight check migrations/0042_drop_legacy_status.sql --repo .
```

Or as a GitHub Action:

```yaml
- uses: <your-username>/preflight-action@v1
  with:
    migration-path: migrations/
```

See [`docs/quickstart.md`](docs/quickstart.md) for connecting a real repo and enabling data sampling.

## API Overview

| Endpoint | Method | Purpose |
|---|---|---|
| `/v1/runs` | POST | Submit a migration + repo ref for analysis |
| `/v1/runs/{id}` | GET | Poll run status and results |
| `/v1/repos` | POST | Register a repo (source path, DB dialect, sampling config) |
| `/v1/repos/{id}/history` | GET | Past runs for a repo |

Full API reference: [`docs/api.md`](docs/api.md)

## Project Structure

```
preflight/
├── api/            # FastAPI routers, schemas — thin HTTP shell
├── core/           # framework-agnostic engine: parsing, shadow orchestration, tracing, risk
├── workers/        # arq worker + analysis tasks
├── cli/            # `preflight check` CLI entrypoint
├── github_action/  # GitHub Action wrapper
├── models/, migrations/, repositories/, services/, shared/
├── tests/          # unit / integration / property / load
└── docker/
```

`core/` has zero FastAPI dependency by design — the CLI and GitHub Action call it directly without needing the API running.

## Testing

```bash
pytest tests/unit
pytest tests/integration   # requires docker-compose services running
pytest tests/property      # correctness checks for the code tracer's column resolution
```

## Roadmap

- [x] Raw SQL migration parsing (sqlglot)
- [x] Shadow DB orchestration + real lock measurement
- [x] Static code tracer (Python / SQLAlchemy)
- [ ] Alembic / Django migration file support
- [ ] MySQL dialect support
- [ ] Hosted dashboard + run history UI

## Known Limitations

Static resolution of ORM calls to columns is inherently approximate in highly dynamic code (dynamic attribute access, string-built queries). Unresolved call sites are reported explicitly as **"unknown"** rather than silently omitted — see [`docs/algorithm-notes.md`](docs/algorithm-notes.md) for measured precision/recall against real repos.

## License

MIT — see [LICENSE](LICENSE).
