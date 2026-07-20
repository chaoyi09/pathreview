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
