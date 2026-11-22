# Quick start — DM-assistant / Delivery Insights

Минимальный путь: поднять стек, подключиться к XTracker, получить отчёт.
Все команды проверены на живых досках XTracker (2026‑11‑22).

---

## 1. Что нужно до запуска

| Что | Зачем | Где взять |
|---|---|---|
| JWT XTracker | доступ к MCP‑мосту Nebula | личный токен в трекере |
| `XTRACKER_MCP_URL` | адрес MCP‑моста | в `.env.example` уже прописан стенд |
| Ключ очереди (`queue_key`) | что анализируем | напр. `VITRINA`, `PLSPRINT` |
| Docker + Compose | самый быстрый запуск | — |

---

## 2. Запуск через Docker (рекомендуется)

```bash
cp .env.example .env
# откройте .env и заполните:
#   XTRACKER_MCP_TOKEN=<ваш JWT>
#   GRAFANA_ADMIN_PASSWORD=<любой пароль>
docker compose up --build
```

Сервисы и порты из [`docker-compose.yml`](docker-compose.yml):

| Сервис | Порт | Назначение |
|---|---|---|
| frontend | `http://localhost:8084` | лендинг (nginx, проксирует API) |
| backend | `http://localhost:8000` | FastAPI, `/api/health`, `/api/analyze` |
| grafana | `http://localhost:3001` | дашборды по истории анализов (SQLite) |

Откройте `http://localhost:8084`, вставьте ссылку на доску трекера и ключ очереди.

---

## 3. Запуск без Docker

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

export XTRACKER_MCP_URL="https://nebula-staging.x5.ru/..."
export XTRACKER_MCP_TOKEN="<ваш JWT>"
uvicorn main:app --host 0.0.0.0 --port 8000
```

Для транспорта через локальный CLI дополнительно нужен бинарь `xt` и профиль
с API‑ключом (`xt auth status` должен быть зелёным) — см. §4.

---

## 4. Два транспорта данных: `TRACKER_TRANSPORT`

Оба транспорта реализуют один и тот же контракт клиента, поэтому отчёт
не зависит от того, как получены данные.

| `TRACKER_TRANSPORT` | Как ходим в трекер | Когда использовать |
|---|---|---|
| `mcp` (по умолчанию) | MCP‑мост Nebula, персональный JWT | штатный режим, работает отовсюду, где есть egress |
| `cli` | локальный бинарь `xt` (subprocess, JSON) | когда из пода/машины нет egress до MCP‑моста |

```bash
# транспорт по умолчанию — mcp; менять не нужно
# переключение на CLI:
export TRACKER_TRANSPORT=cli
xt auth status        # профиль должен быть авторизован
```

Что важно знать про `cli`:

* `REQUIRE_OWN_TOKEN` в режиме `cli` игнорируется — JWT пользователя не нужен,
  авторизация живёт в профиле `xt`;
* `mcp_url` в запросе `/api/analyze` с `cli` не используется (вернётся `400`,
  это защита от путаницы, а не сбой);
* если `xt` не найден — backend стартует, но анализ вернёт понятную ошибку.

---

## 5. Проверка живости

```bash
curl -s http://localhost:8000/api/health | python3 -m json.tool
```

В ответе смотрите `tracker_transport` (какой транспорт активен) и
`require_own_token`.

Демо‑отчёт без трекера (для проверки UI):

```bash
curl -s http://localhost:8000/api/demo | python3 -m json.tool | head -40
```

---

## 6. Анализ очереди через API

```bash
curl -s -X POST http://localhost:8000/api/analyze \
  -H 'Content-Type: application/json' \
  -d '{"queue_key": "VITRINA"}' | python3 -m json.tool | head -60
```

Только POST, используйте `curl -X POST` (GET вернёт `405`).
Поля запроса — `queue_key` либо `board_url` (id доски можно взять из адреса
`https://xtracker.x5team.ru/agile/board/<id>`).

---

## 7. Локальные проверки (без сети к трекеру)

Все скрипты лежат в `backend/` и запускаются напрямую:

```bash
python3 backend/verify_ttm.py            # ТТМ/lead-time: 153 / 153
python3 backend/verify_all_rules.py      # правила, статусы, гигиена доски
python3 backend/verify_offline.py        # расчёты на офлайн‑фикстурах
python3 backend/verify_stage_chain.py    # цепочка этапов: 18 / 18
node --check frontend/app.js             # синтаксис фронта
```

Запускайте их после правок в `backend/analysis.py`, `backend/status_map.py`,
`backend/ttm.py` — это быстрый способ поймать регрессию до выката.

---

## 8. Частые проблемы

| Симптом | Причина | Что делать |
|---|---|---|
| `401` / `403` на `/api/analyze` | нет/просрочен JWT | вставьте свежий токен в форму или в `.env` |
| `400 ... MCP URL не используется` | включён `cli`, а передан `mcp_url` | уберите `mcp_url` из запроса |
| `xt` не найден | в `PATH` нет бинаря | `export PATH="$HOME/.local/bin:$PATH"` |
| Grafana требует пароль при старте | не задан `GRAFANA_ADMIN_PASSWORD` | задайте его в `.env` |
| Пустые Waiting Time / Flow Efficiency | нет истории статусов | это ожидаемо: значения копятся со снапшотов в SQLite |

---

## Ссылки

* [`README.md`](README.md) — что умеет система, архитектура, метрики.
* [`docs/architecture.drawio`](docs/architecture.drawio) — схема архитектуры.
