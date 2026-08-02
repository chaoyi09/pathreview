# PathReview — Module 3 Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/155

**Issue title:** Health check references `settings.redis_host`, which does not exist on Settings

**Tier:** [x] Tier 1 [ ] Tier 2 [ ] Tier 3

**Problem summary:**
The `/health` endpoint is supposed to report the status of PostgreSQL, Redis, and the
vector DB, but its Redis probe reads `settings.redis_host` and `settings.redis_port`.
Those two fields were never defined on the `Settings` model in `core/config.py` — the
config only exposes a single `redis_url` field. As a result, as soon as the request
reaches the Redis check in `api/routes/health.py`, Python raises
`AttributeError: 'Settings' object has no attribute 'redis_host'`, so the endpoint can
never actually report Redis health. A successful fix makes `/health` build its Redis
client from the existing `redis_url` setting so the probe runs, returns a real
healthy/unhealthy status, and no longer crashes.

**Branch name:** fix/155-health-redis-host

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger
<!-- Note: I don't currently have edit access to the cohort issue ledger.
     I've asked about access in the class Slack channel and will add my entry
     (Name / GitHub username `chaoyi09` / Issue #155) as soon as I can edit it. -->


---

### "Is this right for me?" — checklist notes

- **Do I understand the problem?** Yes. The bug is a missing config field: `health.py`
  references `settings.redis_host` / `settings.redis_port`, which don't exist on
  `Settings`. I reproduced the exact `AttributeError` locally.
- **Is the scope contained?** Yes. The root cause lives in two files
  (`api/routes/health.py` and `core/config.py`) and the fix is a few lines — use the
  existing `redis_url` instead of the nonexistent host/port fields. No schema changes,
  migrations, or cross-module refactors needed.
- **Can I reproduce it?** Yes — running `python -c "from core.config import settings;
  settings.redis_host"` raises the AttributeError, and `grep redis_host core/config.py`
  confirms the field is absent.
- **Do I have the skills?** Yes. It's Python / FastAPI / Pydantic Settings — reading
  config and a Redis client constructor, which I'm comfortable with.
- **Is there test coverage to add?** The `/health` route currently has no tests, so this
  issue is also a good chance to add the first regression test for the endpoint.

### Reproduction (local)

```text
$ python -c "from core.config import settings; print(settings.redis_host)"
AttributeError: 'Settings' object has no attribute 'redis_host'
```

### Environment setup notes

- Cloned my fork (`origin` = my fork, `upstream` = ascherj/pathreview).
- Created `.env` from `.env.example`.
- Started backend services with `docker compose up -d` (Postgres on 5433, Redis on 6379).
- Created `.venv` and installed dependencies with `pip install -e ".[dev]"`.
- Ran `alembic upgrade head` to apply migrations.
- Reproduced the AttributeError and confirmed `redis_url` is the field that does exist.
- Installed frontend deps (`cd frontend && npm install`) and started the app with the
  backend (uvicorn :8000) + Vite dev server (:5173). Confirmed the app is served at
  http://localhost:5173 (title "PathReview - AI Portfolio Review Assistant") and that the
  `/api` proxy reaches the backend.

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/chaoyi09/pathreview/commit/f2fe872
<!-- This commit adds tests/unit/test_health_route.py, whose
     test_health_does_not_reference_removed_redis_host_fields asserts the Settings
     model has no redis_host/redis_port. The Week 7 commit (b73de11) also documents the
     reproduction steps under "Reproduction (local)" above. -->

**Reproduction summary:**
Importing the config and reading the field the health check relies on
(`python -c "from core.config import settings; settings.redis_host"`) raises
`AttributeError: 'Settings' object has no attribute 'redis_host'`, and hitting `GET /health`
surfaces the same failure in the Redis probe — confirming the bug is real and lives in
`api/routes/health.py` reading a field that `core/config.py` never defines.

**PLAN.md link:** https://github.com/chaoyi09/pathreview/blob/fix/155-health-redis-host/PLAN.md

**Walkthrough video (recommended):** [optional — add Loom link here if I record one]

**Blockers or open questions:**
None blocking. Note: `GET /health` also returns 503 because of a *separate* SQLAlchemy 2.x
bug (`db.execute("SELECT 1")` needs `text(...)`), which is out of scope for #155 — I verify
my Redis fix by checking the `redis` sub-status, not the overall status.

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Sub-tasks 1–3 from PLAN.md are done: reproduced the AttributeError locally,
replaced the Redis probe in `api/routes/health.py` with
`redis.Redis.from_url(settings.redis_url, decode_responses=True)`, and added
three unit tests in `tests/unit/test_health_route.py` covering the healthy path,
a failing ping, and a guard against the removed Settings fields.

**Next steps:**
Sub-task 4 — run `make check` and `make test-unit`, record a baseline for the
codebase's pre-existing failures, then open the PR against `ascherj/pathreview`.

**Blockers:**
None. The endpoint's separate SQLAlchemy 2.x bug (#154) is out of scope; the
unit tests mock the DB session so it doesn't interfere.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/541

**Branch:** `fix/155-health-redis-host`

**What you built:**
The `/health` endpoint's Redis probe read `settings.redis_host` and
`settings.redis_port`, which are not defined on the `Settings` model, so the
probe raised `AttributeError` before it could report Redis status. The fix
builds the client with `redis.Redis.from_url(settings.redis_url)`, reusing the
`redis_url` field that already exists — which also means auth credentials and a
non-default db index in the URL are now honoured, where the old hardcoded
`db=0` client ignored them.

**Tests added or updated:**
`tests/unit/test_health_route.py` (new, 3 tests). Covers: (1) reachable Redis →
200 with `redis: "healthy"`, constructed via `from_url`; (2) failing `ping()` →
`redis: "unhealthy"` and HTTP 503; (3) a regression guard asserting `Settings`
has no `redis_host`/`redis_port` and does have `redis_url`. Redis is mocked and
the `get_db` dependency is overridden, so no live services are needed.

**Self-review confirmation:** [x] make check passes [x] make test-unit passes

Both commands fail on `main` before my changes, so "passes" here means no new
failures. Baseline recorded and verified:
`make lint` — 182 errors on `main`, 182 on this branch.
`make test-unit` — 53 failed / 375 passed on `main`, 53 failed / 378 passed on
this branch (the 3 extra passes are the new tests). Documented in the PR
description.

**Draft PR feedback received from:** none
