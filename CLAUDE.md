# VAPT Tool — Project Context for Claude Code

> **This file is auto-loaded by Claude Code at the start of every session in this repo.**
> It is the single source of truth for the project: architecture, stack, schemas, file layout, and the exact build sequence to follow. Read this fully before generating any code.

---

## ⚡ 5 Rules That Override Everything Else

Claude Code reads this block first. These rules are non-negotiable and take precedence over any seemingly reasonable deviation:

1. **Follow the file structure in Section 3 exactly.** If a path isn't listed there, ask before creating it.
2. **Every scanning module MUST emit the normalized finding schema from Section 4.3 — exactly.** Wrong or missing keys cause silent data loss at the aggregator. This is the pipeline's only internal contract.
3. **The Ollama system prompt in Section 4.5 is byte-for-byte.** Do not paraphrase, reorder, or "improve" it — the JSON output shape depends on its exact wording.
4. **Never weaken the safety guardrails in Section 9.** Authorization checks, private-IP rejection, and non-destructive-only payloads are non-negotiable. Ask before relaxing anything, even for convenience during testing.
5. **Finish and verify one step before starting the next.** Later steps assume earlier contracts are correct (Step 6's aggregator trusts Steps 3–5's schema; Step 7's PDF trusts Step 6's JSON shape). Skipping ahead corrupts the pipeline silently.

---

## 0. What You Are Building

An **Automated Vulnerability Assessment and Penetration Testing (VAPT) tool** — locally hosted, air-gapped, built for IIT Kanpur Computer Centre under the supervision of Navpreet Singh.

The tool accepts a target domain, runs five scanning modules in parallel, sends the aggregated findings to a local LLM (Ollama + Qwen 2.5 7B) for CVSS scoring and remediation generation, then produces a professional PDF report plus a live web dashboard.

**Project metadata:** Prepared by Aayush Yadav (B.Tech CSE, Bennett University) · 15-day build sprint · Classification: Confidential — Internal Use Only.

### Key Design Principles
- **Air-gapped & local** — no data leaves the IITK network; no external API calls from the tool itself.
- **Parallel execution** — all five scanning modules run concurrently via Celery workers, not sequentially.
- **AI-enhanced** — Ollama running Qwen 2.5 7B on local GPU provides intelligent vulnerability analysis instead of raw tool dumps.
- **Zero operational cost** — every component is open-source.
- **Authorized-only** — the tool requires explicit authorization confirmation before any scan is dispatched.

### Scope
- Network reconnaissance — open ports, services, subdomains, DNS records, WHOIS
- Web application scanning — active vulnerability detection via OWASP ZAP and Nikto
- SSL/TLS configuration — certificate validity, weak ciphers, protocol versions
- HTTP security headers — CSP, CORS, X-Frame-Options, HSTS, cookie security flags
- OWASP Top 10 checks — SQL injection, reflected XSS, IDOR, path traversal, open redirect, error disclosure

---

## 1. System Architecture

A six-layer pipeline. Each layer has one clear responsibility and talks to its neighbors through a well-typed interface.

| Layer | Component | Responsibility |
|---|---|---|
| 1 — Input | React Frontend | Domain entry form, authorization checkbox, live scan status display |
| 2 — Backend | FastAPI + PostgreSQL | Request validation, job creation, status API, report delivery |
| 3 — Queue | Celery + Redis | Async job dispatch, parallel worker orchestration, task state management |
| 4 — Scanning | 5 Python modules | Execute external security tools, normalize output to shared JSON schema |
| 5 — Intelligence | Ollama + Qwen 2.5 7B | CVSS scoring, risk ranking, natural-language remediation generation |
| 6 — Output | WeasyPrint + React | PDF report generation, interactive vulnerability dashboard |

### Data Flow (end to end)
1. User submits domain via React form with authorized-scope confirmation.
2. FastAPI validates domain format and creates a scan record in PostgreSQL (`status: queued`).
3. FastAPI serializes the scan job and pushes it to the Redis queue.
4. A Celery worker picks up the job and dispatches five parallel subtasks.
5. Each scanning module executes its tools via `subprocess` and returns normalized JSON.
6. The Results Aggregator merges, deduplicates, and sorts all findings by severity.
7. The aggregated JSON is sent to Ollama (Qwen 2.5 7B) for AI analysis.
8. Ollama returns structured findings with CVSS scores and remediation steps.
9. WeasyPrint renders a PDF report; PostgreSQL is updated (`status: complete`).
10. The React frontend polls for completion and displays the dashboard.

---

## 2. Technology Stack

| Category | Technology | Version | Purpose |
|---|---|---|---|
| Backend API | FastAPI | 0.111+ | REST API, async handling, OpenAPI docs |
| Task Queue | Celery | 5.3+ | Distributed task queue for parallel scan execution |
| Message Broker | Redis | 7.2+ | Job queue broker + Celery result backend |
| Database | PostgreSQL | 16+ | Persistent storage for scan records and reports |
| ORM | SQLAlchemy + Alembic | 2.0+ | Models and schema migrations |
| Recon Tools | nmap, subfinder | latest | Port scanning, service detection, subdomain enumeration |
| Web Scanner | OWASP ZAP, Nikto | latest | Active web vulnerability scanning |
| SSL Scanner | testssl.sh, sslscan | latest | TLS configuration and certificate analysis |
| AI Engine | Ollama + Qwen 2.5 7B | latest | Local LLM for CVSS scoring and remediation |
| PDF Generator | WeasyPrint | 60+ | HTML-to-PDF rendering |
| Template Engine | Jinja2 | 3.1+ | HTML report templates |
| Frontend | React 18 + Tailwind CSS | latest | Single-page application |
| Charts | Recharts | 2.12+ | Severity charts and heatmaps |
| Containerization | Docker + Compose | latest | Reproducible deployment |
| Language | Python | 3.11+ | Primary backend language |

**Hardware:** GPU with ≥8 GB VRAM for the Ollama layer. Qwen 2.5 7B at Q4_K_M quantization uses ~4.5 GB VRAM. Reference dev machine: AMD Ryzen 9, NVIDIA RTX 4060 8 GB, Ubuntu 24.04.

---

## 3. Project File Structure

Build toward this exact layout. Create directories as needed — don't pre-scaffold empty folders, create them as part of each step below.

```
vapt-tool/
├── backend/
│   ├── main.py                  # FastAPI app entry point
│   ├── config.py                # Environment config (DB URL, Redis, Ollama)
│   ├── models.py                # SQLAlchemy ORM models
│   ├── schemas.py               # Pydantic request/response schemas
│   ├── database.py              # DB session and engine setup
│   ├── routers/
│   │   ├── scan.py              # POST /scan, GET /scan/{id}/status
│   │   └── report.py            # GET /scan/{id}/report (PDF download)
│   ├── tasks/
│   │   ├── celery_app.py        # Celery app configuration
│   │   ├── base_task.py         # Shared helpers (status updates, normalization)
│   │   ├── scan_orchestrator.py # Main Celery task: dispatches all 5 subtasks
│   │   ├── recon.py             # Celery task: nmap + subfinder + WHOIS + DNS
│   │   ├── webscan.py           # Celery task: OWASP ZAP + Nikto
│   │   ├── ssl_tls.py           # Celery task: testssl.sh + sslscan
│   │   ├── headers.py           # Celery task: HTTP security headers
│   │   └── owasp.py             # Celery task: SQLi, XSS, IDOR, traversal, etc.
│   ├── analysis/
│   │   ├── aggregator.py        # Merges + deduplicates 5 module outputs
│   │   └── ollama_client.py     # Ollama API integration
│   ├── reports/
│   │   ├── generator.py         # WeasyPrint PDF generation
│   │   └── templates/
│   │       └── report.html      # Jinja2 PDF template
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── pages/
│   │   │   ├── Home.jsx         # Domain input form
│   │   │   ├── ScanStatus.jsx   # Live progress polling
│   │   │   └── Report.jsx       # Vulnerability dashboard
│   │   └── components/
│   │       ├── SeverityBadge.jsx
│   │       └── VulnTable.jsx
│   └── package.json
├── docker-compose.yml
├── CLAUDE.md                    # This document
└── README.md
```

---

## 4. Component Deep Dives

### 4.1 FastAPI Backend

Backbone of the system: exposes a REST API consumed by the React frontend, handles domain validation, creates scan records in PostgreSQL, pushes jobs to Redis/Celery, returns job IDs for async polling.

**Key Endpoints**

| Method | Endpoint | Description | Response |
|---|---|---|---|
| POST | `/api/scan` | Submit a new scan. Validates domain, creates DB record, enqueues Celery task | `{ job_id, status: 'queued' }` |
| GET | `/api/scan/{id}/status` | Poll scan progress. Module statuses + completion % | `{ status, modules, progress }` |
| GET | `/api/scan/{id}/report` | Download completed PDF report | `application/pdf` binary |
| GET | `/api/scan/{id}/findings` | Fetch structured findings JSON for dashboard | `{ findings[], summary }` |
| GET | `/api/health` | Healthcheck for Docker/monitoring | `{ status: 'ok' }` |

**Validation logic (before dispatching any scan):**
- Domain format validation via Python's `validators` library — rejects IP ranges, `localhost`, internal hostnames.
- Authorization flag — request body must include `authorized: true`, else reject with HTTP 403.
- Duplicate detection — if an active scan for the same domain exists within the last 10 minutes, return the existing job ID instead of starting a new one.

**`backend/routers/report.py`** — one endpoint:
- `GET /api/scan/{id}/report` — fetch the `Report` record from PostgreSQL; return `pdf_data` as a `StreamingResponse` with `media_type='application/pdf'` and header `Content-Disposition: attachment; filename=vapt_report_{domain}_{date}.pdf`. Return HTTP 404 if no report exists yet (scan still running). Return HTTP 202 with `{ "status": "pending" }` if scan is complete but PDF generation is still in progress.

### 4.2 Redis + Celery Job Queue

Redis is the broker between FastAPI and Celery. FastAPI never executes scans itself — it serializes the job and pushes it to Redis, returning the job ID immediately (`202 Accepted`) while the scan runs in the background.

```
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
CELERY_TASK_SERIALIZER = 'json'
CELERY_TASK_SOFT_TIME_LIMIT = 300   # 5 min per module
CELERY_TASK_TIME_LIMIT = 360        # hard kill at 6 min
CELERY_WORKER_CONCURRENCY = 5       # one worker per scanning module
```

> **Dev shortcut (Steps 1–8 only, remove before Docker):** Set `CELERY_TASK_ALWAYS_EAGER = True` in `celery_app.py` to run tasks synchronously in-process without needing Redis — useful for testing individual modules in isolation via a plain Python call. Remove this flag before Step 9.

Workers listen to Redis continuously. A job triggers `scan_orchestrator.py`, which uses a **Celery group** to dispatch all five scanning subtasks in parallel, with a **chord** callback firing once all five complete — triggering aggregation.

### 4.3 Scanning Engine

Five independent Celery tasks. Each wraps one or more external tools via `subprocess` with a controlled timeout, and all return findings in the **same normalized schema**:

> **Temp file cleanup:** scanning modules write to `/tmp/{tool}_{scan_id}.*`. Each module must delete its own temp files in a `finally` block after parsing, or they accumulate and fill disk on long-running instances.

```json
{
  "module": "recon",
  "tool": "nmap",
  "type": "open_port",
  "title": "Port 22 (SSH) open",
  "evidence": "22/tcp open ssh",
  "severity": "Info",
  "cvss": 0.0,
  "target": "example.com",
  "found_by": ["recon"]
}
```

> **Field notes:**
> - `severity` and `cvss` are placeholders — the AI layer overwrites them.
> - `found_by` starts as a single-element list `["module_name"]`. The aggregator extends it to `["recon", "webscan"]` if the same finding is detected by multiple modules. **Every module must include this field** — don't omit it, the aggregator's dedup logic depends on it.

#### 4.3.1 Recon Module — `backend/tasks/recon.py`
Passive/semi-active info gathering only — no exploit payloads, just attack-surface mapping.

| Tool | Flags | Finds |
|---|---|---|
| nmap | two-phase: `--top-ports 100` (run to completion) then best-effort `-p- --host-timeout 60s`, both `-sV -sC --open -T4 --min-rate 1000` | Open ports, service versions (the security-relevant part). **OS fingerprint (`-O`) deliberately disabled** — requires root/raw sockets; Celery worker runs unprivileged. OS field stays empty; no crash. Accepted as-is: OS is informational-only, unreliable behind CDN/load-balancers, and doesn't feed any finding's CVSS score. Enable by adding `cap_add: [NET_RAW, NET_ADMIN]` to the worker container in docker-compose (Step 9) if OS detection is ever required. |
| subfinder | `-d {domain} -silent` | Subdomains via passive DNS (no brute-force). **Depth limited without API keys** — see subfinder note below. |
| whois | `whois {domain}` | Registrar, registration date, nameservers, abuse contacts |
| dnspython | A, MX, TXT, NS, CNAME lookups | DNS records, SPF/DMARC/DKIM presence |

#### 4.3.2 Web Scan Module — `backend/tasks/webscan.py`
OWASP ZAP (headless daemon, primary) + Nikto (fast misconfig pattern-match, 6,700+ signatures).

| Tool | Mode | Finds |
|---|---|---|
| OWASP ZAP | Headless daemon + active scan + ajax spider | XSS, SQLi, CSRF, broken auth, insecure redirects |
| Nikto | `-h {target} -Format json -Tuning 1234578` | Outdated software, dangerous files, misconfigs |

ZAP controlled via `python-owasp-zap-v2.4`: start daemon → wait for ready → spider + active scan → retrieve alerts as JSON → kill process. Total timeout: 5 minutes.

> **Port isolation:** ZAP binds to a port (default 8090). If two scans run concurrently, they will collide. Use a per-scan port derived from `scan_id`, e.g. `8090 + (hash(scan_id) % 900)`, and pass it as `-port {port}` to the daemon and to the `ZAPv2` proxies dict.

#### 4.3.3 SSL/TLS Module — `backend/tasks/ssl_tls.py`

| Tool | Flags | Finds |
|---|---|---|
| testssl.sh | `--jsonfile /tmp/ssl.json {target}` | Protocol versions, ciphers, BEAST/POODLE/HEARTBLEED, cert chain, HSTS |
| sslscan | `--xml=/tmp/ssl2.xml {target}` | SSL version support, preferred ciphers, DH params, OCSP |

Key checks: TLS 1.0/1.1 enabled (High), self-signed cert (High), expired cert (Critical), weak RC4/DES ciphers (High), missing HSTS (Medium), insecure renegotiation (High).

#### 4.3.4 HTTP Headers Module — `backend/tasks/headers.py`
Lightest module — single GET request, pure Python (`requests`), no external tools.

| Header | Expected | Severity if missing/wrong |
|---|---|---|
| Content-Security-Policy | Present and restrictive | Medium |
| X-Frame-Options | DENY or SAMEORIGIN | Medium (clickjacking) |
| Strict-Transport-Security | `max-age >= 31536000; includeSubDomains` | High |
| X-Content-Type-Options | nosniff | Low |
| Referrer-Policy | strict-origin or no-referrer | Low |
| Permissions-Policy | Restricts camera/mic/geolocation | Low |
| CORS (Access-Control-Allow-Origin) | Not `*` on sensitive endpoints | High if wildcard |
| Set-Cookie flags | Secure; HttpOnly; SameSite=Strict | Medium if missing |

#### 4.3.5 OWASP Top 10 Module — `backend/tasks/owasp.py`
Targeted, **non-destructive** active tests (read-only payloads only, no data modification).

| Check | Method | Indicator |
|---|---|---|
| SQL Injection | Inject `' OR '1'='1` in params/fields | SQL error or boolean-based response diff |
| Reflected XSS | Inject `<script>alert(1)</script>` in all params | Payload reflected unsanitized |
| IDOR | Enumerate numeric IDs (`/user/1`, `/user/2`) | Different user data for sequential IDs |
| Path Traversal | Inject `/../../../etc/passwd` in file params | File content or FS errors in response |
| Open Redirect | Inject external URL in redirect params | 302 to injected external domain |
| Error Disclosure | Send malformed requests | Stack traces / DB errors / version info in 500s |

### 4.4 Results Aggregator — `backend/analysis/aggregator.py`

Triggered by the Celery chord callback once all five tasks complete.

1. Collect all findings from the Celery group result set.
2. Deduplicate on `(type, evidence)` — same vuln found by multiple tools merges into one finding with all tool names listed.
3. Enrich with OWASP Top 10 category mapping based on finding type.
4. Sort by preliminary severity (Critical > High > Medium > Low > Info).
5. Truncate evidence strings to 500 chars to avoid oversizing the Ollama prompt.
6. Return final JSON payload ready for the Ollama client.

### 4.5 Ollama AI Analysis Layer — `backend/analysis/ollama_client.py`

Ollama runs locally on GPU, serving Qwen 2.5 7B Instruct, API at `localhost:11434`.

**Exact system prompt (use this text verbatim):**
```
You are a professional cybersecurity analyst. You will be given raw vulnerability
findings from an automated VAPT scan in JSON format. Your task is to analyze
these findings and return ONLY valid JSON with no markdown, no explanation,
and no text outside the JSON object.

For each finding, provide:
- title: concise vulnerability name
- description: what this vulnerability is and why it matters
- severity: one of Critical / High / Medium / Low / Informational
- cvss_score: CVSS v3.1 base score (0.0-10.0)
- cvss_vector: CVSS v3.1 vector string
- owasp_category: OWASP Top 10 2021 category if applicable
- cve_reference: most relevant CVE if known, else null
- evidence: the most significant evidence snippet
- remediation: specific, actionable remediation steps
- priority: 1 (fix immediately) to 5 (fix when convenient)

Return: { executive_summary, risk_score (0-100), findings[], total_critical,
total_high, total_medium, total_low, total_informational }
```

**Model config:**
```
model: qwen2.5:7b
format: json           # forces valid JSON output
temperature: 0.1       # low temp for consistent, factual analysis
num_predict: 4096
num_ctx: 8192
```

User message format: `f'Analyze these VAPT findings for {domain}: {json.dumps(aggregated)}'`

**Error handling:** if Ollama times out (120s) or returns invalid JSON, fall back to a rule-based severity analysis built from the raw aggregated findings — never let the pipeline hard-fail here.

### 4.6 Report Generation — `backend/reports/generator.py`

WeasyPrint converts a Jinja2-rendered HTML template to PDF (inline CSS only, no external deps, so it renders identically offline). Stored as `BYTEA` in PostgreSQL, served via `GET /api/scan/{id}/report`.

**Report sections:**
- Cover page — target domain, scan date, risk score badge, IITK Computer Centre header
- Executive Summary — non-technical overview, overall risk rating, top 3 critical issues
- Severity Breakdown — color-coded count table with CVSS distribution
- Findings Catalogue — one card per vulnerability: title, severity badge, CVSS score, OWASP category, CVE reference, evidence snippet, remediation steps
- Technical Appendix — raw scanner output summary, scan configuration, tool versions
- Footer — confidentiality notice, scan timestamp, page numbers

Risk score badge color: red if >70, amber if >40, green otherwise.

### 4.7 React Frontend — `frontend/src/`

Single-page app, communicates exclusively with the FastAPI backend.

| Page | Component | Description |
|---|---|---|
| Home | `Home.jsx` | Domain input, authorization checkbox, submit button. Client-side URL validation before POSTing to `/api/scan` |
| Scan Status | `ScanStatus.jsx` | Polls `GET /api/scan/{id}/status` every 3s. Live progress bar per module (Recon, Web scan, SSL, Headers, OWASP), color-coded states |
| Report Dashboard | `Report.jsx` | Recharts severity bar chart, sortable/filterable vulnerability table, expandable per-finding cards, PDF download button |

---

## 5. Data Schemas

### POST `/api/scan` — Request Body
```json
{ "domain": "example.com", "authorized": true, "notes": "Optional scope notes" }
```

### GET `/api/scan/{id}/status` — Response
```json
{
  "job_id": "uuid",
  "domain": "example.com",
  "status": "running",
  "progress": 60,
  "started_at": "ISO8601",
  "modules": {
    "recon": "complete",
    "webscan": "running",
    "ssl_tls": "complete",
    "headers": "complete",
    "owasp": "queued"
  }
}
```

### GET `/api/scan/{id}/findings` — Response
```json
{
  "executive_summary": "string",
  "risk_score": 72,
  "total_critical": 2,
  "total_high": 5,
  "total_medium": 8,
  "total_low": 4,
  "total_informational": 11,
  "findings": [
    {
      "title": "TLS 1.0 Enabled",
      "severity": "High",
      "cvss_score": 7.5,
      "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N",
      "owasp_category": "A02:2021 - Cryptographic Failures",
      "cve_reference": "CVE-2014-3566",
      "evidence": "TLS 1.0 accepted on port 443",
      "remediation": "Disable TLS 1.0 and 1.1 in server config.",
      "priority": 2,
      "module": "ssl_tls"
    }
  ]
}
```

---

## 6. Database Schema (PostgreSQL)

| Table | Column | Type | Description |
|---|---|---|---|
| scans | id | UUID (PK) | Unique scan job identifier |
| scans | domain | VARCHAR(255) | Target domain submitted |
| scans | status | ENUM | queued / running / analysing / complete / failed |
| scans | authorized | BOOLEAN | Authorization confirmation flag |
| scans | started_at | TIMESTAMP | When Celery picked up the job |
| scans | completed_at | TIMESTAMP | When AI analysis finished |
| scans | module_statuses | JSONB | Per-module status map |
| scans | raw_findings | JSONB | Aggregated findings before AI analysis |
| scans | ai_analysis | JSONB | Full Ollama response incl. CVSS scores |
| scans | risk_score | INTEGER | Overall risk score 0-100 |
| reports | id | UUID (PK) | Report identifier |
| reports | scan_id | UUID (FK) | References `scans.id` |
| reports | pdf_data | BYTEA | PDF binary content |
| reports | generated_at | TIMESTAMP | PDF generation timestamp |

---

## 7. Docker Compose Architecture

Six services: PostgreSQL, Redis, Ollama (GPU passthrough), FastAPI backend, Celery worker, React frontend (via Nginx).

```yaml
services:
  postgres:
    image: postgres:16-alpine
  redis:
    image: redis:7-alpine
  ollama:
    image: ollama/ollama       # GPU passthrough enabled
  backend:
    build: ./backend           # FastAPI image
  worker:
    build: ./backend           # same image, different entrypoint
    command: celery -A tasks.celery_app worker --loglevel=info -c 5
  frontend:
    build: ./frontend          # React + Nginx
```

Ollama GPU access:
```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```

Per-service detail (used in Step 9 below):
1. **postgres** — `postgres:16-alpine`, env `POSTGRES_DB=vapt POSTGRES_USER=vapt POSTGRES_PASSWORD=vapt_secure_2025`, volume `postgres_data`, healthcheck `pg_isready`
2. **redis** — `redis:7-alpine`, healthcheck `redis-cli ping`
3. **ollama** — `ollama/ollama:latest`, volume `ollama_data:/root/.ollama`, GPU reservation as above; post-start step should auto-pull `qwen2.5:7b` if not already present
4. **backend** — build `./backend`, depends on postgres/redis/ollama, env `DATABASE_URL REDIS_URL OLLAMA_URL`, ports `8000:8000`, command `uvicorn main:app --host 0.0.0.0 --port 8000 --reload`, volume `./backend:/app`
5. **worker** — same image as backend, depends on postgres/redis, command `celery -A tasks.celery_app worker --loglevel=info -c 5`
6. **frontend** — build `./frontend`, ports `3000:80`, depends on backend; Dockerfile uses `node:20-alpine` build stage + `nginx:alpine` serve stage

**backend/Dockerfile:** `FROM python:3.11-slim`, install system packages (`nmap nikto sslscan whois curl wget`), install `testssl.sh` from GitHub to `/usr/local/bin/testssl.sh`, install the `subfinder` binary, `COPY requirements.txt`, `WORKDIR /app`. **`whois` is required** — `backend/tasks/recon.py` shells out to the `whois` binary (bounded subprocess); without it WHOIS recon is skipped (logged, non-fatal).

**.env.example:** `DATABASE_URL`, `REDIS_URL`, `OLLAMA_URL`, `SECRET_KEY`, `ALLOWED_HOSTS`

**README.md:** quick-start — `git clone` → copy `.env.example` → `docker-compose up` → open `localhost:3000`. Note that the first run pulls Qwen 2.5 7B (~4.5 GB). Include the authorized-use-only legal disclaimer.

---

## 8. 15-Day Development Timeline (reference pacing — not a hard schedule for Claude Code)

| Day | Focus | Deliverable | Est. hrs |
|---|---|---|---|
| 1 | Project setup | Docker Compose skeleton, FastAPI entry point, PostgreSQL models, Alembic migrations, `.env` config | 5–6 |
| 2 | Celery + Redis | Celery app config, scan orchestrator, group/chord pattern, status polling | 4–5 |
| 3 | Recon module | nmap wrapper, subfinder, WHOIS parser, DNS lookup, normalized output | 5–6 |
| 4 | Web scan module | ZAP daemon setup + client, Nikto subprocess, alert normalization | 6–7 |
| 5 | SSL/TLS module | testssl.sh integration, sslscan XML parser, cipher mapper | 4–5 |
| 6 | Headers module | Header fetcher, per-header policy checker, CORS/cookie checks | 3–4 |
| 7 | OWASP module | SQLi/XSS/IDOR/traversal/open-redirect testers | 5–6 |
| 8 | Aggregator | 5-module merger, dedup, OWASP mapper, severity pre-sort | 3–4 |
| 9 | Ollama AI layer | Client, prompt engineering, JSON parsing, CVSS extraction | 5–6 |
| 10 | PDF report | Jinja2 template, WeasyPrint integration, badges, storage | 5–6 |
| 11 | React frontend | Domain form, auth checkbox, API client, status polling | 5–6 |
| 12 | Dashboard UI | Recharts chart, vuln table, filter/sort, expandable cards | 5–6 |
| 13 | End-to-end test | Full scan against test target, bug fixes, edge cases | 6–7 |
| 14 | Docker + deployment | `docker-compose up`, Nginx config, hardening, healthchecks | 4–5 |
| 15 | Demo prep | README, demo flow, sample report, summary slide | 3–4 |

---

## 9. Security & Ethical Guardrails (non-negotiable, build these in from Step 1)

- **Authorization** — every scan request requires `authorized: true`; log every authorization with a timestamp.
- **Network isolation** — domain validation must reject RFC 1918 private ranges (`10.x.x.x`, `192.168.x.x`, `172.16-31.x.x`) and `localhost`.
- **Non-destructive testing** — all active tests use read-only payloads only. No write operations, no data modification, no DoS payloads, ever.
- **Audit trail** — every scan (target, timestamp, operator) is permanently logged in PostgreSQL.
- **Rate limiting** — max 3 concurrent scans; reject duplicate active scans for the same domain.
- **Data privacy** — zero external API calls; all analysis stays local; no scan data leaves the machine.

This tool exists for **authorized assessments only**. Unauthorized scanning is illegal (IT Act 2000, India, and equivalent statutes elsewhere). It's built for educational/research use within IITK infrastructure. When in doubt about a feature request, default to the more conservative, more clearly-authorized-only behavior.

### Approved test targets for development/demo
| Target | URL | Type | Notes |
|---|---|---|---|
| testphp.vulnweb.com | http://testphp.vulnweb.com | Intentionally vulnerable PHP app | Best for web scan + OWASP module testing |
| HackTheBox Labs | lab.hackthebox.com (VPN) | CTF-style machines | Requires account + active VPN |
| Local DVWA | http://localhost:80 (Docker) | Self-hosted | Safest: `docker run -d dvwa/dvwa` |

Never scan anything else without prior written authorization.

---

## 10. Working Agreement for Claude Code in This Repo

- Follow the file structure in Section 3 exactly — don't introduce alternate layouts.
- Every scanning module must emit the normalized finding schema from Section 4.3, including the `found_by` field — this is the contract the aggregator depends on.
- Keep the system prompt in Section 4.5 byte-for-byte when implementing `ollama_client.py`.
- Don't weaken the safety guardrails in Section 9 (authorization checks, private-IP/localhost rejection, non-destructive-only payloads, rate limiting) even if a later step seems to need it relaxed for convenience — ask first.
- Prefer finishing and testing one step before starting the next, since later steps assume earlier contracts are correct.
- Use `config.py` settings everywhere — never hardcode URLs, ports, or credentials.

## 🚫 Common Mistakes — Don't Do This

- **Hardcoding `localhost:6379` or `localhost:11434`** — always use `config.REDIS_URL` and `config.OLLAMA_URL`.
- **Running nmap or ZAP in the main FastAPI process** — heavy tools always run inside Celery tasks with timeouts. Never block the API server.
- **Letting the ZAP daemon linger after a scan** — kill it in a `finally:` block, not just after the happy path. A leaked ZAP process will block the port for the next scan.
- **Using `verify=True` in the headers module** — most test targets have self-signed or invalid certs. Use `verify=False` with `urllib3.disable_warnings()`, not as a security decision but as a scanner design decision.
- **Omitting `found_by` from module output** — the aggregator's dedup logic reads this field. If it's missing, dedup silently breaks and every finding from every module appears as a separate duplicate.
- **Storing PDF bytes in `scans.ai_analysis` JSONB** — PDF data goes in `reports.pdf_data` (BYTEA), not in the scans table. Storing binary in JSON will corrupt the column.
- **Binding ZAP to port 8090 unconditionally** — concurrent scans collide. Derive the port from `scan_id` (see Section 4.3.2 note).
- **Not cleaning up `/tmp/{tool}_{scan_id}.*` files** — each module must delete its temp files in a `finally:` block or disk fills during long test sessions.
- **Calling `ollama_client.analyse()` before aggregation** — Ollama must receive the aggregated, deduplicated findings, not raw per-module output. The token count matters.

---

## 11. Build Sequence — Run These Prompts In Order

Work through these one at a time in this Claude Code session. Each step assumes the previous ones are done — the schema, file layout, and module contracts defined above are the contract every step must follow. **Review and verify after each step before moving to the next** (verification checklist in Section 12).

### STEP 1 — Project Setup & FastAPI Skeleton
Create, with complete production-ready code:
1. `backend/config.py` — Pydantic `BaseSettings` from `.env`: `DATABASE_URL`, `REDIS_URL`, `OLLAMA_URL` (default `http://localhost:11434`)
2. `backend/database.py` — SQLAlchemy engine, `SessionLocal`, `Base`, `get_db` dependency
3. `backend/models.py` — ORM models:
   - `Scan`: id (UUID PK), domain, status (Enum: queued/running/analysing/complete/failed), authorized (bool), started_at, completed_at, module_statuses (JSON), raw_findings (JSON), ai_analysis (JSON), risk_score (int), notes (str)
   - `Report`: id (UUID PK), scan_id (FK), pdf_data (LargeBinary), generated_at
4. `backend/schemas.py` — Pydantic schemas: `ScanRequest` (domain: HttpUrl, authorized: must be True, notes: optional), `ScanResponse` (job_id, status, domain), `ScanStatus` (job_id, domain, status, progress 0-100, modules dict, started_at), `FindingSchema` (title, severity, cvss_score, owasp_category, cve_reference, evidence, remediation, priority, module)
5. `backend/main.py` — FastAPI app, CORS enabled for `localhost:3000`, include routers from `routers/scan.py` and `routers/report.py`
6. `backend/routers/scan.py` — `POST /api/scan` (validate → create DB record → push to Celery → return job_id), `GET /api/scan/{id}/status`, `GET /api/scan/{id}/findings`
   `backend/routers/report.py` — `GET /api/scan/{id}/report` (StreamingResponse of PDF bytes; 404 if no report; 202 if scan pending) — see Section 4.1 for full spec
7. `backend/requirements.txt` — pinned versions. Use at minimum:
   ```
   fastapi==0.111.0
   uvicorn[standard]==0.29.0
   celery==5.3.6
   redis==5.0.4
   psycopg2-binary==2.9.9
   sqlalchemy==2.0.30
   alembic==1.13.1
   pydantic-settings==2.2.1
   requests==2.31.0
   urllib3==2.2.1
   python-whois==0.9.4
   dnspython==2.6.1
   python-owasp-zap-v2.4==0.0.21
   validators==0.22.0
   weasyprint==61.2
   jinja2==3.1.4
   python-multipart==0.0.9
   ```
8. `alembic.ini` + `migrations/env.py` configured for the models above

After creating all files: `cd backend && pip install -r requirements.txt --break-system-packages`

### STEP 2 — Celery Configuration & Scan Orchestrator
1. `tasks/celery_app.py` — Celery app named `vapt`, Redis broker/backend from config, concurrency 5, JSON serializer, soft/hard time limits 300s/360s, autodiscover `['tasks.recon','tasks.webscan','tasks.ssl_tls','tasks.headers','tasks.owasp']`
2. `tasks/scan_orchestrator.py` — `@app.task` `scan_orchestrator(scan_id, domain)`: set status `running` → Celery group of the 5 subtasks → chord callback `aggregate_and_analyse` → update `module_statuses` as subtasks progress → on subtask failure mark that module `failed` but continue others → after chord: call aggregator → ollama_client → reports/generator → set status `complete`
3. `tasks/base_task.py` — `BaseTask` with `update_module_status(scan_id, module_name, status)` and `normalize_finding(module, tool, type, title, evidence, severity='Info')` returning the standard finding dict

   All five scanning modules import from here:
   ```python
   from tasks.base_task import BaseTask, normalize_finding, update_module_status
   ```
   The `normalize_finding` helper must always include `found_by=[module]` in the returned dict.

### STEP 3 — Recon Module
`backend/tasks/recon.py`. Receives `scan_id`, `domain`. Must run all of:
1. **nmap**: **two-phase scan** (see note below) → parse XML for open ports/services/versions/OS → one finding per open port (`type='open_port'`)
2. **subfinder**: `-d {domain} -silent -o /tmp/sub_{scan_id}.txt` → one finding per subdomain (`type='subdomain_found'`)
3. **WHOIS**: `python-whois` → registrar, creation_date, expiry_date, name_servers (deduplicated) as informational; flag expiry within 90 days as Medium
4. **DNS**: `dnspython` → A/MX/TXT/NS/CNAME; check missing SPF (TXT), missing DMARC (`_dmarc` TXT), missing DKIM → each missing record is a Medium finding

Error handling: tool failure/timeout → log as failed finding, continue with remaining tools. Update `module_statuses`: `recon -> running` at start, `recon -> complete` at end.

> **nmap two-phase scan — deliberate deviation from the original single `-p-` / "60s nmap" spec (decided & empirically tuned during Step 3 build):**
> A single full `-p-` scan cannot return results against a filtered/CDN host (e.g. Vercel, which filters every port except 80/443). nmap must wait out no-response probes on all ~65k filtered ports, and two things go wrong: (a) a hard `subprocess` SIGKILL leaves an empty/truncated XML → zero findings; (b) **`--host-timeout`, when it fires mid port-scan, makes nmap ABANDON the host and report zero ports too** (verified — it is *not* a partial-results mechanism for a single host). The only scan that reliably reports on such a host is one small enough to **run to completion**.
> So nmap runs in two phases, merged & de-duplicated by port:
> - **Phase 1 — `--top-ports 100`, NO `--host-timeout`, `subprocess(timeout=130)`.** Allowed to finish; captures the services that actually matter (web/ssh/mail/db). Instant on a normal host; ~2 min on a fully-filtered host but it completes and reports (verified: found 80/443 on Vercel in ~120s).
> - **Phase 2 — `-p- --host-timeout 60s`, `subprocess(timeout=70)`.** Best-effort extra coverage: a normal host finishes in seconds and adds high-port services; a filtered host times out harmlessly because Phase 1 already has the real ports.
> - **Adaptive skip:** Phase 2 is skipped entirely when Phase 1 took >30s (a slow Phase 1 means the host is filtered, so `-p-` can't complete anyway and would only waste ~70s). This bounds nmap to **≤130s**.
>
> Both phases use `-sV -sC --open -T4 --min-rate 1000`.

> **Recon timing budget — per-task limit raised to soft 600s / hard 660s (overrides the global 300/360):** recon runs its tools *sequentially* in one task, so every external call is hard-bounded. After subfinder was raised to 60s (API sources) and the nmap app-ports phase was added, recon's worst case grew to ~356s — past the default 300s soft limit. Rather than cripple it, recon gets a generous per-task ceiling (free: webscan's ~430s gates total scan time, and recon's internal budgets cap the real work regardless):
>
> | Stage | Worst case | Bound |
> |---|---|---|
> | nmap | 240s | Phase 1 `--top-ports 100` subprocess cap 180s (run-to-completion on filtered hosts) + Phase 2b app-ports cap 60s. Phase 2a full `-p-` (≤70s) only runs on responsive hosts (Phase 1 <30s), so the 240s and 100s branches are mutually exclusive — 240s is the bound. |
> | subfinder | 60s | `subprocess(timeout=60)` — raised from 30s to accommodate API sources (GitHub token etc.) |
> | WHOIS | 20s | `subprocess(['whois', domain], timeout=20)` — SIGKILL-enforced; raw text parsed via `whois.parser.WhoisEntry.load`. **Never call `python_whois.whois()` directly — it has no timeout and a hung WHOIS server stalls recon.** |
> | DNS | 36s | `resolver.timeout/lifetime = 4`; 9 queries max (5 records + DMARC + 3 DKIM selectors; SPF reuses the TXT answers — no extra query) |
> | **Total** | **~356s** | **244s margin under the 600s soft limit** |
>
> Per-task limits are set per module by runtime, NOT globally: fast modules (headers, owasp) keep the tight 300/360 so a hang is caught quickly; only the legitimately-slow modules get raised ceilings — **recon 600/660, webscan 480/540** (see webscan note). No downstream/schema impact — recon runs in parallel and is never the critical path (ZAP dominates). Every stage degrades to "no findings" on failure rather than raising; the top-level `run_recon` try/except is the final backstop (marks the module `failed`, returns partial findings).

> **subfinder API keys — free options to improve subdomain coverage (deferred, add when available):**
> Without keys subfinder only queries free/public sources and finds fewer subdomains. These sources are free and significantly improve depth:
> | Source | Free tier | Get key at |
> |---|---|---|
> | Chaos (ProjectDiscovery) | Completely free | chaos.projectdiscovery.io |
> | GitHub | Free with account | github.com → Settings → Developer settings → Personal access tokens |
> | VirusTotal | 1,000 req/day free | virustotal.com → API key |
> | SecurityTrails | 50 req/month free | securitytrails.com |
> | Censys | Free tier | censys.io |
> | WhoisXMLAPI | 500 req/month free | whoisxmlapi.com |
>
> To wire them in: add keys to `.env` and write them to `~/.config/subfinder/provider-config.yaml` (subfinder's config format) in the Docker entrypoint. Chaos + GitHub are the highest-value free pair — add those first.

### STEP 4 — Web Scan Module (OWASP ZAP + Nikto)
`backend/tasks/webscan.py`. Receives `scan_id`, `domain`, `target_url = f'https://{domain}'`.
1. **OWASP ZAP**: `pip install python-owasp-zap-v2.4` → start daemon `subprocess(['zap.sh','-daemon','-port','8090','-config','api.disablekey=true','-config','connection.timeoutInSecs=60'])` → poll `http://localhost:8090/JSON/core/view/version/` up to 60s → `ZAPv2(apikey='', proxies={'http':'http://127.0.0.1:8090'})` → `zap.spider.scan(target_url)` (poll to completion) → `zap.ascan.scan(target_url)` (poll to completion) → `zap.core.alerts(baseurl=target_url)` → normalize each alert (use ZAP risk as severity) → kill ZAP process
2. **Nikto**: `subprocess(['nikto','-h',target_url,'-Format','json','-o',f'/tmp/nikto_{scan_id}.json','-Tuning','1234578b','-maxtime','120s'])` → parse JSON → normalize each item

Error handling: if ZAP fails to start within 60s, skip it and continue with Nikto only. Update `module_statuses`: `webscan -> running` then `webscan -> complete`.

> **webscan timing — per-task limit override (decided during Step 4 build):** webscan's worst case is ZAP wait (≤60s) + ZAP spider+ascan (≤240s) + Nikto (≤130s) = **~430s**, which exceeds the default Celery 300s soft / 360s hard limit — the task would be SIGKILL'd mid-scan, breaking the chord and failing the *entire* scan. Since ZAP active scanning is the pipeline's intended long pole ("ZAP's ~5 min dominates"), `run_webscan` overrides the per-task limits to **`soft_time_limit=480, time_limit=540`** (the decorator overrides the global config for this task only; other modules keep 300/360). Internal budgets (`_ZAP_READY_TIMEOUT=60`, `_ZAP_SCAN_BUDGET=240`, `_NIKTO_TIMEOUT=130`) keep the real worst case ~430s, a ~50s margin under the raised soft limit. This does **not** affect the orchestrator: `scan_orchestrator` dispatches the chord and returns immediately, so it never blocks on webscan's runtime.

### STEP 5 — SSL/TLS, Headers, and OWASP Modules
Three files:

**`backend/tasks/ssl_tls.py`** — receives `scan_id`, `domain`; `target = f'https://{domain}'`
- testssl.sh: `subprocess(['testssl.sh','--jsonfile',f'/tmp/ssl_{scan_id}.json','--quiet','--color','0',domain])` → parse JSON → map CRITICAL→Critical, HIGH→High, WARN→Medium
- sslscan: `subprocess(['sslscan',f'--xml=/tmp/sslscan_{scan_id}.xml',domain])` → parse XML → flag SSLv3 (High), TLS 1.0 (High), RC4 (High), self-signed (High), expired cert (Critical), weak DH <2048 bits (High)

**`backend/tasks/headers.py`** — receives `scan_id`, `domain`
- `requests.get(f'https://{domain}', timeout=15, verify=False, allow_redirects=True)`
- Findings for: missing CSP (Medium), missing X-Frame-Options (Medium), missing HSTS (High), missing X-Content-Type-Options (Low), CORS `*` (High), cookie missing Secure (Medium), missing HttpOnly (Medium), missing SameSite (Low)
- Also record all present headers as Informational findings

**`backend/tasks/owasp.py`** — receives `scan_id`, `domain`; `target = f'https://{domain}'`. Implement 5 non-destructive test functions using `requests`:
1. `test_sqli(target)` — inject `' OR 1=1--` and `'` into GET params; check for SQL errors
2. `test_xss(target)` — inject `<script>alert(VAPT)</script>` into params; check if reflected
3. `test_path_traversal(target)` — try `/../../../etc/passwd` in path params
4. `test_open_redirect(target)` — try `?next=https://evil.com` in redirect params
5. `test_error_disclosure(target)` — send malformed requests, check for stack traces in 500s

Run all 5, collect findings, 30s timeout per function.

### STEP 6 — Results Aggregator + Ollama AI Analysis
Two files:

**`backend/analysis/aggregator.py`** — `aggregate(findings_list: List[List[dict]]) -> dict`
1. Flatten all 5 module lists into one
2. Deduplicate on `(type, evidence[:100])`, merging into one finding with all module names in a `found_by` list
3. OWASP mapping via lookup dict
4. Sort Critical → High → Medium → Low → Informational
5. Truncate evidence to 500 chars
6. Return `{ findings: [...], total: n, scan_metadata: { timestamp, tool_versions } }`

**`backend/analysis/ollama_client.py`** — `analyse(aggregated: dict, domain: str) -> dict`
1. Build the system prompt — **exact text from Section 4.5 above**
2. User message: `f'Analyze these VAPT findings for {domain}: {json.dumps(aggregated)}'`
3. `POST http://localhost:11434/api/chat` with `model='qwen2.5:7b', format='json', stream=False, options={'temperature':0.1,'num_predict':4096,'num_ctx':8192}`, messages = system + user
4. Parse `response.json()['message']['content']` as JSON
5. Validate required keys exist (`executive_summary`, `findings`, `risk_score`)
6. Return parsed dict

Error handling: Ollama timeout (120s) or invalid JSON → fall back to a rule-based analysis built from the raw aggregated findings.

### STEP 7 — PDF Report Generator
`backend/reports/generator.py` + `backend/reports/templates/report.html`

**generator.py** — `generate_pdf(scan: Scan, analysis: dict) -> bytes`
1. `env = Environment(loader=FileSystemLoader('reports/templates')); template = env.get_template('report.html')`
2. Render with `domain`, `scan_date`, `risk_score`, `executive_summary`, `findings`, `total_critical/high/medium/low`, `iitk_logo_text='IIT Kanpur Computer Centre'`
3. `weasyprint.HTML(string=html).write_pdf()`
4. Return bytes

**report.html** — professional Jinja2 template with:
- Cover section: IITK header, domain, date, risk score badge (red >70, amber >40, green otherwise)
- Executive summary block
- Severity count table, color-coded cells
- Per-finding card: severity badge, CVSS score, OWASP category, CVE reference if present, monospace evidence, bulleted remediation
- Print-optimized CSS (`@page` rules, `page-break-inside: avoid` on finding cards), Arial/Helvetica fonts
- Footer: "Confidential — IIT Kanpur Computer Centre — Authorized Use Only"

### STEP 8 — React Frontend
`frontend/src/`. Stack: React 18, React Router v6, Tailwind, Recharts, Axios.

- **App.jsx** — routes: `/` (Home), `/scan/:id/status` (ScanStatus), `/scan/:id/report` (Report)
- **pages/Home.jsx** — domain input + URL validation; authorization checkbox ("I confirm I am authorized to test this target") gating the submit button; POST `/api/scan`; redirect to `/scan/{job_id}/status` on success; friendly error on 403/400
- **pages/ScanStatus.jsx** — poll `/api/scan/{id}/status` every 3s; overall progress bar; 5 module rows (Recon, Web Scan, SSL/TLS, Headers, OWASP Top 10) each with a colored status chip (gray=queued, blue=running, green=complete, red=failed); redirect to `/scan/{id}/report` on complete
- **pages/Report.jsx** — fetch `/api/scan/{id}/findings` on mount; summary cards (Critical red, High orange, Medium yellow, Low blue); Recharts BarChart by severity; sortable VulnTable (Severity badge, Title, CVSS, OWASP Category, Module, Priority); click row to expand evidence/CVE/remediation; "Download PDF" button hitting `/api/scan/{id}/report`
- **components/SeverityBadge.jsx** — colored pill: Critical=red, High=orange, Medium=yellow, Low=blue, Info=gray

### STEP 9 — Docker Compose & Final Setup
At project root, create:
- **docker-compose.yml** — the 6 services exactly as specified in Section 7 above (postgres, redis, ollama, backend, worker, frontend) with healthchecks and dependencies
- **backend/Dockerfile** — as specified in Section 7
- **.env.example** — `DATABASE_URL`, `REDIS_URL`, `OLLAMA_URL`, `SECRET_KEY`, `ALLOWED_HOSTS`
- **README.md** — quick-start instructions + legal disclaimer, as specified in Section 7

---

## 12. How to Verify Each Step Actually Works

Generated code can look complete and still be broken. Don't move to the next step until the current one passes its checks below. "Files exist" is never enough on its own — always get to at least the command-line or browser check.

### After Step 1 — FastAPI Skeleton
1. Files exist: `backend/main.py`, `config.py`, `models.py`, `schemas.py`, `database.py`, `routers/scan.py`, `routers/report.py`, `requirements.txt`
2. Dependencies install cleanly:
   ```bash
   cd backend && pip install -r requirements.txt --break-system-packages
   ```
   No red error text — warnings are fine, errors are not.
3. Server actually starts:
   ```bash
   uvicorn main:app --reload
   ```
   Look for `Uvicorn running on http://127.0.0.1:8000` with no traceback.
4. Open `http://127.0.0.1:8000/docs` in a browser — FastAPI's auto-generated interactive page should list `/api/scan`, `/api/scan/{id}/status`, `/api/scan/{id}/findings`, `/api/health`.
5. Click "Try it out" on `/api/health` → Execute. Expect `{"status": "ok"}`.
6. **Expected to fail at this stage:** `/api/scan` will error out because PostgreSQL doesn't exist yet (no Docker until Step 9). That's normal — don't try to fix it now.

### After Step 2 — Celery + Redis
You need Redis running locally to test this before Docker exists. Easiest: `docker run -d -p 6379:6379 redis:7-alpine` (just Redis alone, not the whole stack).
1. Start a worker:
   ```bash
   celery -A tasks.celery_app worker --loglevel=info
   ```
   It should print recognized tasks (`tasks.recon`, `tasks.webscan`, etc.) and "celery@... ready" with no crash.
2. From a Python shell, manually trigger the orchestrator task and watch the worker terminal log it picking up the job and dispatching subtasks (they'll fail at this point — recon.py etc. don't exist yet — but you should see the *dispatch* happen).

### After Step 3 — Recon Module
1. Confirm `nmap`, `subfinder`, `whois` are actually installed and on PATH (`which nmap`, etc.) — if not, this will silently fail or hang.
2. Test the module directly, isolated from Celery, against an approved target (Section 9 list):
   ```python
   from tasks.recon import run_recon  # or whatever it's named
   results = run_recon("scan_id_test_123", "testphp.vulnweb.com")
   print(results)
   ```
3. Check the output is a **list of dicts**, each matching the normalized schema from Section 4.3 (`module`, `tool`, `type`, `title`, `evidence`, `severity`, `cvss`, `target` — all keys present, no `None` where a string is expected).
4. Sanity-check the content makes sense: open ports for a known test site shouldn't come back empty or as garbage strings.

### After Step 4 — Web Scan Module (ZAP + Nikto)
1. Confirm ZAP and Nikto are installed and runnable standalone first — `nikto -Version`, and check ZAP starts via `zap.sh -daemon` without you touching Python yet.
2. This module is the slowest and flakiest — test with a generous timeout and watch for the ZAP daemon actually responding at `http://localhost:8090/JSON/core/view/version/` before the scan starts (curl it manually if unsure).
3. Run the module directly (same pattern as Step 3) against `testphp.vulnweb.com`, confirm it returns normalized findings and that the ZAP process actually gets killed afterward (`ps aux | grep zap` should show nothing lingering).

### After Step 5 — SSL/TLS, Headers, OWASP Modules
1. Confirm `testssl.sh` and `sslscan` are installed and runnable standalone.
2. Test each of the three modules independently (`ssl_tls.py`, `headers.py`, `owasp.py`) the same way as Step 3, against an HTTPS-enabled approved target.
3. For `headers.py` specifically — this one's pure Python with no external tool, so it should be near-instant and easy to eyeball: confirm the findings make sense against what you can see by checking response headers yourself (e.g. via browser dev tools Network tab on the same site).
4. For `owasp.py` — confirm it's only sending GET-style read-only payloads (re-read the actual code, don't just trust the prompt was followed) and that it completes within the 30s-per-function timeout.

### After Step 6 — Aggregator + Ollama Analysis
1. Before touching Ollama: feed the aggregator a small fake list of findings (copy-paste a few sample dicts) and confirm dedup/sort/OWASP-mapping logic works in isolation — this avoids burning Ollama calls debugging plain Python logic.
2. Confirm Ollama itself is running and the model is pulled:
   ```bash
   ollama list   # should show qwen2.5:7b (or your chosen tag)
   curl http://localhost:11434/api/chat -d '{"model":"qwen2.5:7b","messages":[{"role":"user","content":"say hello in JSON: {\"msg\": ...}"}],"format":"json","stream":false}'
   ```
   You should get back valid JSON, not an error or empty response.
3. Run `ollama_client.analyse()` with real aggregated output from Steps 3–5. Check the returned dict actually has `executive_summary`, `risk_score`, `findings` — and that `findings` isn't empty if real vulnerabilities were found upstream.
4. Deliberately test the fallback: temporarily stop Ollama (`systemctl stop ollama` or kill the process) and confirm your code falls back to rule-based scoring instead of crashing the whole pipeline.

### After Step 7 — PDF Report
1. Call `generate_pdf()` directly with a sample `analysis` dict (can be fake/hand-written data) and confirm it returns actual PDF bytes (`pdf_bytes[:4] == b'%PDF'` is a quick sanity check).
2. Write those bytes to a file and **actually open the PDF** — check the cover page, risk badge color, severity table, and that finding cards aren't overlapping or cut off mid-page.
3. Try it with zero findings and with a large number of findings (10+) to make sure the layout doesn't break at either extreme.

### After Step 8 — React Frontend
1. ```bash
   cd frontend && npm install && npm run dev
   ```
   Confirm it compiles without errors and opens in the browser.
2. Manually walk the flow: enter a domain on Home, confirm the submit button stays disabled until the authorization checkbox is ticked, submit, confirm you land on the ScanStatus page.
3. Since the backend may not be fully wired yet, it's fine if polling shows errors — but check the *error handling* itself works (a friendly message, not a blank crashed page).
4. Once Step 1–7 are all working together, do a real end-to-end run and confirm the Report page actually renders the bar chart, the table is sortable, and the PDF download button works.

### After Step 9 — Docker Compose
1. ```bash
   docker-compose up --build
   ```
   Watch the logs — all 6 services should report healthy/started, no restart loops (a service stuck restarting means its healthcheck or startup command is broken).
2. Run `docker-compose ps` — every service should show `Up` (or `healthy`), not `Restarting` or `Exited`.
3. Open `http://localhost:3000` — the full app should load.
4. Do one full real scan end-to-end against an approved test target through the actual UI, and confirm a PDF downloads at the end. This is the real "does the whole project work" test — everything before this was a piece, this is the whole machine.
5. Check disk usage didn't balloon: `docker system df`.
