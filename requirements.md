# AI News Collector — Detailed Requirements

## 1. Overview

A single-user local web application that scrapes configurable RSS news sources, uses an LLM to score collected articles against a configurable set of topics, and generates an editable markdown newsletter containing the top 10 most relevant articles.

The application is intended to be run locally on the user's machine. No authentication, no multi-tenancy.

## 2. Goals & Non-Goals

### Goals
- Let the user trigger a scrape of configured RSS sources from the UI.
- Show live progress of an ongoing scrape.
- Automatically score each collected article against configurable topics using an LLM.
- Generate a newsletter draft (markdown) containing the top 10 articles by score.
- Let the user edit the draft in the UI and save it.
- Let the user manage sources and topics from the UI.

### Non-Goals
- No user accounts, auth, or multi-user support.
- No newsletter history (only one current draft is stored).
- No email delivery / no publishing workflow.
- No WYSIWYG rich-text editor (markdown textarea only).
- No full-article body scraping beyond what the RSS feed provides.
- No scheduled/cron scraping (manual trigger only).

## 3. Functional Requirements

### FR-1: Scrape Sources
- **FR-1.1** The user can start a scrape by clicking a "Start Scrape" button on the Dashboard.
- **FR-1.2** While a scrape is running, the button is disabled and a status panel is shown.
- **FR-1.3** Only one scrape may run at a time. A second start attempt returns HTTP 409.
- **FR-1.4** A scrape iterates over every enabled source and fetches its RSS feed.
- **FR-1.5** For each RSS entry, the scraper extracts: `title`, `url`, `summary` (or description), `published_at`, `source_id`.
- **FR-1.6** Articles are deduplicated by `url` across the entire database. An article already present is skipped (not re-inserted and not re-scored — see FR-3.4).
- **FR-1.7** Per-source failures (network error, malformed feed) are logged and do NOT abort the run; the run continues with the next source.
- **FR-1.8** On run completion, the run's `scrape_runs` row is marked `done` with `finished_at` set. On unrecoverable failure (e.g. missing LLM API key), the run is marked `error` with an error message.

### FR-2: Watching Ongoing Scrape (Polling)
- **FR-2.1** The frontend polls `GET /api/scrape/status` every ~1.5 seconds while a scrape is known to be running.
- **FR-2.2** The status response includes: `state` (`idle` | `running` | `done` | `error`), `sources_total`, `sources_done`, `articles_found`, `articles_scored`, `last_log` (a human-readable one-liner, e.g. `"Source 2/3: news.ycombinator.com — 18 articles, 12 scored"`), `started_at`, `finished_at` (nullable), `error` (nullable).
- **FR-2.3** The UI displays the `last_log` line and the counters; no persistent log file is required.
- **FR-2.4** When `state` transitions to `done`, polling stops and the article table refreshes.

### FR-3: LLM Relevance Scoring
- **FR-3.1** For each newly-inserted article, the backend calls the LLM to score its relevance against the current list of topics.
- **FR-3.2** The LLM is prompted with the article title + summary + topic list, and is asked to return structured JSON: `{"score": <integer 0-100>, "reason": "<short string>"}`.
- **FR-3.3** The returned `score` is persisted on the article row. On JSON parse failure or LLM error, one retry is attempted; if still failing, `score` is set to `0`, a short error is logged, and the run continues.
- **FR-3.4** Re-scoring existing articles (e.g., when topics change) is out of scope for the initial version. Only newly-inserted articles are scored.
- **FR-3.5** Scoring happens inline during a scrape run, so `articles_scored` can be reported in progress updates.

### FR-4: Newsletter Draft Generation
- **FR-4.1** The user clicks "Generate Draft" on the Newsletter page to trigger `POST /api/newsletter/generate`.
- **FR-4.2** The backend selects the top 10 articles ordered by `score DESC, published_at DESC`.
- **FR-4.3** The backend makes a single LLM call with a structured prompt that includes the 10 articles' titles, urls, and summaries, asking for a markdown newsletter: a short intro paragraph followed by 10 sections — each section containing the article title (as a link), source name, and a 2–3 sentence blurb.
- **FR-4.4** The returned markdown replaces the current saved draft in the single-row `newsletter` table.
- **FR-4.5** If fewer than 10 articles exist, the newsletter includes however many are available (minimum 1). If zero articles exist, generation returns HTTP 400 with a clear message.

### FR-5: Newsletter Editing
- **FR-5.1** The Newsletter page shows a split view: a markdown `<textarea>` on the left and a live-rendered preview on the right.
- **FR-5.2** Preview rendering uses `marked.js` loaded from CDN. No server round-trip for preview.
- **FR-5.3** A "Save" button persists the textarea content via `PUT /api/newsletter`.
- **FR-5.4** On page load, the current saved draft (if any) is fetched via `GET /api/newsletter` and shown. If none exists, the textarea is empty and a placeholder tells the user to click Generate.
- **FR-5.5** Only one newsletter draft exists at any time; re-generating or saving overwrites it.

### FR-6: Source Management
- **FR-6.1** The Settings page shows a list of configured RSS sources with columns: name, url, enabled flag, and a Delete button.
- **FR-6.2** The user can add a new source by entering a name and URL and clicking Add.
- **FR-6.3** The user can delete an existing source.
- **FR-6.4** Sources are persisted in the `sources` table.
- **FR-6.5** On first application run, one default source is seeded: `Hacker News` → `https://news.ycombinator.com/rss`.

### FR-7: Topic Management
- **FR-7.1** The Settings page shows a list of configured scoring topics as free-text strings with a Delete button per entry.
- **FR-7.2** The user can add a new topic via an input and Add button.
- **FR-7.3** The user can delete an existing topic.
- **FR-7.4** Topics are persisted in the `topics` table.
- **FR-7.5** On first application run, the following default topics are seeded:
  - `AI for coding`
  - `LLM coding assistants`
  - `AI developer tools`
  - `Code generation with AI`

## 4. Non-Functional Requirements

- **NFR-1 (Simplicity)** The application must be runnable locally with a single `uvicorn` command after `pip install -r requirements.txt` and a populated `.env`.
- **NFR-2 (Single-user)** No authentication or session management.
- **NFR-3 (Performance)** Polling interval ~1.5s; scrape of the default source (HN RSS, ~30 entries) with scoring should complete in under 2 minutes assuming normal LLM latency.
- **NFR-4 (Resilience)** Per-source and per-article LLM failures must not abort a scrape run.
- **NFR-5 (Config via env)** All LLM connection settings are loaded from `.env` at startup; no secrets in the UI or DB.
- **NFR-6 (Portability)** Must run on Linux and macOS with Python 3.11+.

## 5. Technical Requirements

### 5.1 Stack
- **Language:** Python 3.11+
- **Backend framework:** FastAPI + Uvicorn
- **ORM:** SQLAlchemy 2.x (sync API is fine)
- **Database:** SQLite (`./data.db`), auto-created on first run
- **RSS parsing:** `feedparser`
- **LLM SDK:** `litellm` — calling Anthropic Haiku 4.5 via a custom endpoint
- **Env loading:** `python-dotenv`
- **Frontend:** Plain HTML + vanilla JS + CSS, served by FastAPI as static files from `frontend/`
- **Markdown preview:** `marked.js` loaded from a public CDN

### 5.2 LLM Configuration
- Loaded from `.env` (template provided in `sample.env`):
  - `LLM_PROVIDER=anthropic`
  - `LLM_MODEL=claude-haiku-4-5`
  - `BASE_URL=<custom endpoint>`
  - `LLM_API_KEY=<key>`
- `litellm.completion(...)` is called with `model=LLM_MODEL`, `api_base=BASE_URL`, `api_key=LLM_API_KEY`.
- A startup check logs a warning if `LLM_API_KEY` is missing. Scrape start returns HTTP 500 with a clear message if the key is missing at call time.

### 5.3 Project Layout
```
/workspace/
├── backend/
│   ├── __init__.py
│   ├── main.py              # FastAPI app, static mount, route includes
│   ├── config.py            # .env loading, settings object
│   ├── db.py                # SQLAlchemy engine, session, models
│   ├── scraper.py           # feedparser wrapper, per-source fetch
│   ├── llm.py               # litellm wrapper
│   ├── scoring.py           # article scoring logic + prompt
│   ├── newsletter.py        # draft generation prompt + persistence
│   ├── runner.py            # scrape orchestration + shared run state
│   └── routes/
│       ├── scrape.py
│       ├── articles.py
│       ├── newsletter.py
│       └── settings.py
├── frontend/
│   ├── index.html           # tabs: Dashboard / Newsletter / Settings
│   ├── app.js               # fetch calls, polling, rendering
│   └── styles.css
├── data.db                  # created at runtime (gitignored)
├── requirements.txt
├── sample.env               # provided
└── .env                     # user-created, gitignored
```

### 5.4 Database Schema

**`sources`**
| column | type | notes |
|---|---|---|
| id | INTEGER PK | |
| name | TEXT NOT NULL | |
| url | TEXT NOT NULL UNIQUE | RSS feed url |
| enabled | BOOLEAN NOT NULL DEFAULT 1 | |
| created_at | DATETIME | |

**`topics`**
| column | type | notes |
|---|---|---|
| id | INTEGER PK | |
| text | TEXT NOT NULL UNIQUE | |
| created_at | DATETIME | |

**`articles`**
| column | type | notes |
|---|---|---|
| id | INTEGER PK | |
| source_id | INTEGER FK → sources.id | |
| title | TEXT NOT NULL | |
| url | TEXT NOT NULL UNIQUE | dedupe key |
| summary | TEXT | |
| published_at | DATETIME | may be null |
| score | INTEGER | 0–100, nullable until scored |
| score_reason | TEXT | short string from LLM |
| fetched_at | DATETIME NOT NULL | |

**`scrape_runs`**
| column | type | notes |
|---|---|---|
| id | INTEGER PK | |
| state | TEXT | `running` / `done` / `error` |
| sources_total | INTEGER | |
| sources_done | INTEGER | |
| articles_found | INTEGER | |
| articles_scored | INTEGER | |
| last_log | TEXT | |
| started_at | DATETIME | |
| finished_at | DATETIME | nullable |
| error | TEXT | nullable |

**`newsletter`** (single row, enforced by app logic, id fixed at `1`)
| column | type | notes |
|---|---|---|
| id | INTEGER PK | always `1` |
| content_md | TEXT | current draft markdown |
| updated_at | DATETIME | |

### 5.5 Concurrency Model
- A single module-level `current_run_id: Optional[int]` and `asyncio.Lock` in `runner.py` guard against concurrent scrape starts.
- The scrape job runs as an `asyncio.Task` spawned by the `POST /api/scrape/start` handler. RSS fetches and LLM calls use sync libraries wrapped in `asyncio.to_thread()` so the event loop stays responsive for status polling.

### 5.6 REST API Contract

| Method | Path | Request | Response |
|---|---|---|---|
| POST | `/api/scrape/start` | — | `{"run_id": int}` or 409 |
| GET | `/api/scrape/status` | — | `ScrapeStatus` (see FR-2.2) |
| GET | `/api/articles?limit=50` | query `limit` | `Article[]` sorted by `fetched_at DESC` |
| POST | `/api/newsletter/generate` | — | `{"content_md": str}` |
| GET | `/api/newsletter` | — | `{"content_md": str \| null, "updated_at": str \| null}` |
| PUT | `/api/newsletter` | `{"content_md": str}` | `{"ok": true}` |
| GET | `/api/sources` | — | `Source[]` |
| POST | `/api/sources` | `{"name": str, "url": str}` | `Source` |
| DELETE | `/api/sources/{id}` | — | `{"ok": true}` |
| GET | `/api/topics` | — | `Topic[]` |
| POST | `/api/topics` | `{"text": str}` | `Topic` |
| DELETE | `/api/topics/{id}` | — | `{"ok": true}` |

All responses are JSON. Errors use standard HTTP codes with `{"detail": str}` bodies (FastAPI default).

### 5.7 Prompts (reference)

**Scoring prompt (per article):**
```
You score a news article's relevance to a reader's topics of interest.

Topics of interest:
- {topic_1}
- {topic_2}
...

Article:
Title: {title}
Summary: {summary}

Return ONLY valid JSON of the form:
{"score": <integer 0-100>, "reason": "<one short sentence>"}
```

**Newsletter generation prompt:**
```
You are drafting a short newsletter for a reader interested in these topics:
{topics}

Here are the top {n} articles, pre-selected by relevance:
1. {title} — {url}
   {summary}
...

Write a markdown newsletter with:
- A 1-2 sentence intro
- One section per article, in the same order
- Each section: "### {title}" as a link, then the source name in italics, then a 2-3 sentence blurb summarizing why it matters
- No preamble, no sign-off
```

## 6. Out of Scope (for this version)
- Authentication / multi-user
- Scheduled / recurring scrapes
- Full-text article scraping beyond RSS summary
- Newsletter history, versioning, or export to HTML/PDF/email
- Re-scoring existing articles when topics change
- Deployment / Docker / production hardening
- Automated tests (may be added later; not required for the time-boxed learning implementation)

## 7. Acceptance Criteria
1. Running `uvicorn backend.main:app` with a valid `.env` and opening the frontend in a browser shows the Dashboard with the default source and topics seeded.
2. Clicking "Start Scrape" begins a run; the status panel updates at least once per 2 seconds until completion.
3. After the scrape completes, the article table lists the fetched articles with non-null scores (including `0` for articles where scoring failed).
4. Clicking "Generate Draft" on the Newsletter page returns a markdown newsletter containing up to 10 sections in under ~15 seconds (given normal LLM latency).
5. Editing the textarea updates the rendered preview. Clicking Save persists the edit and reloading the page shows the saved content.
6. Adding or removing a source on the Settings page is reflected in the next scrape run.
7. Adding or removing a topic on the Settings page is reflected in the scoring of newly-inserted articles in the next scrape run.
