# SmartDialer

A working prototype of an outbound collections dialer that keeps agents busy (predictive dialing) without giving up the hard safety guarantees of progressive dialing.

**Core idea:** the pacing logic only *proposes* how many calls to start. An independent, deterministic Safety Controller decides how many are actually allowed, and re-checks inside the database transaction that reserves resources. The statistical estimator controls how much of the safe headroom is used, never how much headroom exists.

```
Campaign -> Pacing Engine -> Safety Controller -> Call Allocator -> Provider -> Events -> PostgreSQL
                                                                                    ^
                                                                 Recovery Sweeper --+
```

**Stack:** Python 3.11, PostgreSQL 16, SQLAlchemy 2.x, pydantic, pytest, threading, FastAPI (dashboard only). No Redis, Kafka, Celery, or ML libraries (see the ADR for why).

**Reviewing this project?** Start with [`docs/PROJECT_REPORT.md`](docs/PROJECT_REPORT.md). It maps every assignment requirement to code and tests.

---

## Setup

Requires Python 3.11+ and Docker.

```bash
docker compose up -d                      # PostgreSQL 16 (creates smartdialer + smartdialer_test)
python -m venv .venv
source .venv/bin/activate                 # Windows: .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python -m smartdialer.db.init_db          # tables, partial indexes, constraints (idempotent)
```

## Run the tests

```bash
pytest -v                                 # full suite, real PostgreSQL
pytest tests/test_simulation.py -v        # fast simulation tests
pytest tests/test_simulation_failures.py -v
```

## Try it

**Live dashboard (best way to see it working):**

```bash
python -m smartdialer.dashboard.live --scenario A --agents 100 --speed 10 --seed 42
# open http://localhost:8000
```

**Simulation (scenarios A-D, three strategies):**

```bash
python -m smartdialer.sim.run --scenario A --strategy predictive_adaptive --providers mixed --seed 42 --agents 100 --duration 120 --step 5.0
python -m smartdialer.sim.run --all --duration 300 --step 5.0 --agents 100 --providers mixed --seed 42   # full matrix
```

**Load test:**

```bash
python -m smartdialer.loadtest.run --agents 100 --workers 4 --rounds 6
python -m smartdialer.loadtest.run --all-scales --workers 8 --rounds 10     # 100 / 1,000 / 10,000 agents
```

**Failure demos (each prints a timeline and asserts invariants):**

```bash
python -m smartdialer.sim.failures.worker_crash
python -m smartdialer.sim.failures.provider_outage
python -m smartdialer.sim.failures.agent_drop
python -m smartdialer.sim.failures.duplicate_event_storm
python -m smartdialer.sim.failures.out_of_order_storm
python -m smartdialer.sim.failures.answer_spike
```

**Live workers (no dashboard):**

```bash
python -m smartdialer.orchestrator.run --workers 4 --campaign 1 --provider mock_a --interval 1.0
```

---

## Code layout

| Package | Responsibility |
|---|---|
| `smartdialer/db/` | SQLAlchemy models, `schema.sql` (partial unique indexes, CHECKs), init script |
| `smartdialer/domain/` | Pure agent and call state machines |
| `smartdialer/allocator/` | Atomic agent + borrower reservation (`FOR UPDATE SKIP LOCKED`) |
| `smartdialer/safety/` | Deterministic Safety Controller (rules R1-R6) |
| `smartdialer/pacing/` | Progressive pacing, Beta-Bayesian estimator, binomial-bound predictive pacing |
| `smartdialer/providers/` | Provider interface, two mock providers, health tracking, circuit breaker |
| `smartdialer/executor/` | Dial execution outside DB transactions |
| `smartdialer/events/` | Event inbox (dedupe) and processor |
| `smartdialer/sweeper/` | Crash and ambiguity recovery |
| `smartdialer/orchestrator/` | Tick loop and worker runner |
| `smartdialer/sim/` | Deterministic simulator, scenarios, invariant checker, failure demos |
| `smartdialer/loadtest/` | Load test and query-plan profiling |
| `smartdialer/dashboard/` | Read-only operator dashboard and audited controls |
| `tests/` | Pytest suite |

## Documentation

| Doc | What it covers |
|---|---|
| [`docs/PROJECT_REPORT.md`](docs/PROJECT_REPORT.md) | **Start here.** Full project report and requirement-to-code-to-test map |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Diagrams: components, state machines, call sequence, concurrency race |
| [`docs/ADR.md`](docs/ADR.md) | Design decisions, trade-offs, guarantees |
| [`docs/SCALING.md`](docs/SCALING.md) | What breaks first from 100 to 100,000 agents, and the fix |
| [`docs/FINAL_ANSWER.md`](docs/FINAL_ANSWER.md) | Short answer to the assignment's final question |
| [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md) | Honest limitations and next steps |
| [`docs/simulation_results.md`](docs/simulation_results.md) | Scenario matrix results |
| [`docs/loadtest_results.md`](docs/loadtest_results.md) | Load test numbers and `EXPLAIN` plans |
| [`docs/DASHBOARD.md`](docs/DASHBOARD.md) | Dashboard guide and 90-second demo script |