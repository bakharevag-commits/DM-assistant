# Quick start

Minimal path: start the stack, connect to your issue tracker, get a report.

---

## 1. Prerequisites

| What | Why | Where to get it |
|---|---|---|
| API session token | access to the issue tracker API | issued by your tracker |
| MCP endpoint URL | address of the MCP bridge | from your deployment (see `.env.example`) |
| Queue key | what to analyze | key of the team queue, e.g. `TEAM` |
| Docker + Compose | fastest way to run | — |

---

## 2. Run with Docker (recommended)

```bash
cp .env.example .env
# open .env and fill in:
#   MCP session token  -> the variable used by your deployment
#   MCP endpoint URL   -> the variable used by your deployment
#   GRAFANA_ADMIN_PASSWORD=<any password>
docker compose up --build
```

Services and ports from [`docker-compose.yml`](docker-compose.yml):

| Service | Port | Purpose |
|---|---|---|
| frontend | `http://localhost:8084` | landing page (nginx, proxies the API) |
| backend | `http://localhost:8000` | FastAPI, `/api/health`, `/api/analyze` |
| grafana | `http://localhost:3001` | dashboards over analysis history (SQLite) |

Open `http://localhost:8084`, paste a board URL and the queue key.

---

## 3. Run without Docker

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# set the MCP endpoint URL and session token env vars used by your deployment
uvicorn main:app --host 0.0.0.0 --port 8000
```

For the local CLI transport you additionally need the tracker CLI binary on
`PATH` and an authenticated CLI profile (`<cli> auth status` must be green) —
see §4.

---

## 4. Two data transports: `TRACKER_TRANSPORT`

Both transports implement the same client contract, so the report does not
depend on how the data was fetched.

| `TRACKER_TRANSPORT` | How it reaches the tracker | When to use |
|---|---|---|
| `mcp` (default) | MCP bridge, personal session token | normal mode; needs network egress to the bridge |
| `cli` | local tracker CLI binary (subprocess, JSON) | when egress to the MCP bridge is unavailable |

```bash
# default transport is mcp; nothing to change
# switch to CLI:
export TRACKER_TRANSPORT=cli
```

What to know about `cli`:

* `REQUIRE_OWN_TOKEN` is ignored in `cli` mode — no user session token is
  needed, authorization lives in the CLI profile;
* `mcp_url` in an `/api/analyze` request is not used with `cli` and returns
  `400` — that is a guard against confusion, not a failure;
* if the CLI binary is missing, the backend still starts and the analysis
  returns a clear error.

---

## 5. Health check

```bash
curl -s http://localhost:8000/api/health | python3 -m json.tool
```

In the response check `tracker_transport` (which transport is active) and
`require_own_token`.

Demo report without a tracker (to check the UI):

```bash
curl -s http://localhost:8000/api/demo | python3 -m json.tool | head -40
```

---

## 6. Analyze a queue via the API

```bash
curl -s -X POST http://localhost:8000/api/analyze \
  -H 'Content-Type: application/json' \
  -d '{"queue_key": "TEAM"}' | python3 -m json.tool | head -60
```

POST only — use `curl -X POST` (GET returns `405`).
Request fields: `queue_key` or `board_url` (the board id can be taken from the
board URL `<tracker-base>/agile/board/<id>`).

---

## 7. Local checks (no network needed)

All scripts live in `backend/` and run directly:

```bash
python3 backend/verify_ttm.py            # TTM / lead time: 153 / 153
python3 backend/verify_all_rules.py      # rules, statuses, board hygiene
python3 backend/verify_offline.py        # calculations on offline fixtures
python3 backend/verify_stage_chain.py    # stage chain: 18 / 18
node --check frontend/app.js             # frontend syntax
```

Run them after touching `backend/analysis.py`, `backend/status_map.py`,
`backend/ttm.py` — the fastest way to catch a regression before rollout.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401` / `403` on `/api/analyze` | missing or expired session token | paste a fresh session token in the form or `.env` |
| `400 ... MCP URL is not used` | `cli` mode with `mcp_url` sent | remove `mcp_url` from the request |
| CLI binary not found | binary not on `PATH` | add its directory to `PATH` |
| Grafana asks for a password at start | `GRAFANA_ADMIN_PASSWORD` not set | set it in `.env` |
| Waiting Time / Flow Efficiency are empty | no status history yet | expected: values accumulate from SQLite snapshots |

---

## Links

* [`README.md`](README.md) — features, architecture, metrics.
* [`docs/architecture.drawio`](docs/architecture.drawio) — architecture diagram.

---

## UI Preview & Demo

To protect corporate data and internal workflows, UI screenshots are available upon request during the interview process.

The interface includes:

* Real-time flow metrics dashboard (TTM, Lead Time, WIP).
* Automated bottleneck and stagnation alerts.
* Personal overload tracking (1 person = 1 active task rule).
