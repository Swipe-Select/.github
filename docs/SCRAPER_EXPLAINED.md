# Web-Scrapper — Complete Technical Deep-Dive

> **Repo:** `Web-Scrapper`  
> **Tech Stack:** Python · Flask · JobSpy (custom fork) · Pandas · Pydantic · BeautifulSoup · TLS-Client  
> **Port:** `5001`  
> **Purpose:** Independent job scraping microservice that aggregates listings from multiple job boards (Indeed, LinkedIn, Rozee.pk) and returns normalized, API-ready JSON.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [How the Server Starts](#2-how-the-server-starts)
3. [The Agent-Based Architecture](#3-the-agent-based-architecture)
4. [The JobSpy Core Engine](#4-the-jobspy-core-engine)
5. [Individual Scraper Agents](#5-individual-scraper-agents)
6. [Data Models & Schemas](#6-data-models--schemas)
7. [The Normalizer](#7-the-normalizer)
8. [Flask Backend API](#8-flask-backend-api)
9. [Filtering & Relevance Logic](#9-filtering--relevance-logic)
10. [Caching System](#10-caching-system)
11. [Resilience & Anti-Bot Features](#11-resilience--anti-bot-features)
12. [API Endpoints Reference](#12-api-endpoints-reference)
13. [Dependencies & Their Roles](#13-dependencies--their-roles)
14. [Design Decisions](#14-design-decisions)
15. [How to Run](#15-how-to-run)
16. [Deployment](#16-deployment)

---

## 1. Project Structure

```
Web-Scrapper/
├── backend/
│   └── app.py                    # Flask API server (the main entry point)
├── jobspy/                       # Custom fork of python-jobspy library
│   ├── __init__.py               # scrape_jobs() — core orchestrator
│   ├── agents.py                 # BaseAgent ABC + JobPostSchema + SearchCriteria
│   ├── model.py                  # Core data models (JobPost, Site, Country, etc.)
│   ├── normalizer.py             # Normalizer class — raw→API-ready transformation
│   ├── config.py                 # User-Agent rotation lists, timeouts, proxies
│   ├── exception.py              # Per-site custom exceptions
│   ├── util.py                   # Session management, salary parsing, helpers
│   ├── indeed/                   # Indeed scraper agent
│   │   ├── __init__.py           # Indeed class (extends Scraper)
│   │   ├── constant.py           # Indeed-specific constants
│   │   └── util.py               # Indeed-specific utilities
│   ├── linkedin/                 # LinkedIn scraper agent
│   │   ├── __init__.py           # LinkedIn class
│   │   ├── constant.py
│   │   └── util.py
│   ├── rozee/                    # Rozee.pk scraper agent
│   │   └── __init__.py           # Rozee class
├── frontend/                     # React dashboard (for testing/visualization)
├── requirements.txt              # Python dependencies
├── pyproject.toml                # Poetry project config
├── render.yaml                   # Render deployment config
└── start_app.sh                  # Startup script
```

---

## 2. How the Server Starts

**File:** `backend/app.py`

```
1. Flask app created
2. CORS enabled for all routes
3. SCRAPER_API_KEY loaded from environment
4. before_request hook checks API key for all routes (except /api/health)
5. Server starts on port 5001 (or $PORT from environment)
6. Available endpoints printed to console
```

**Production (Render):**
```
gunicorn backend.app:app --bind 0.0.0.0:$PORT --workers 1 --threads 4 --timeout 120
```

---

## 3. The Agent-Based Architecture

The scraper uses an **Agent pattern** where each job board is handled by an independent agent. This is defined in `jobspy/agents.py`:

### SearchCriteria (Input Contract)
```python
class SearchCriteria(BaseModel):
    keywords: str                    # "Software Engineer"
    location: str = "Worldwide"
    site_name: list[str]             # ["indeed", "linkedin"]
    results_wanted: int = 20
    is_remote: bool = False
    job_type: str | None = None
    hours_old: int | None = None
```

### JobPostSchema (Output Contract)
```python
class JobPostSchema(BaseModel):
    title: str                       # "Full Stack Developer"
    company: str                     # "Google"
    description: str                 # Cleaned markdown
    location: str                    # "Lahore, Pakistan"
    url: str                         # Unique — used for deduplication
    salaryRange: str | None          # "$80,000 - $120,000/yearly"
    jobType: str | None              # "fulltime"
    source: str                      # "indeed"
    postedAt: str                    # ISO date
    skills: list[str]                # ["Python", "React"]
```

### BaseAgent (Abstract Base Class)
```python
class BaseAgent(ABC):
    def __init__(self, source: str): ...
    
    @abstractmethod
    def scrape(self, criteria: SearchCriteria) -> list[JobPostSchema]: ...
    
    def _harmonize(self, raw_data: dict) -> JobPostSchema: ...
```

Every scraper agent must implement `scrape()` and return a standardized `JobPostSchema` list. This ensures **uniform output** regardless of which job board the data came from.

---

## 4. The JobSpy Core Engine

**File:** `jobspy/__init__.py` — The `scrape_jobs()` function

This is the **orchestrator** that coordinates all scraper agents:

### Step-by-Step Flow:

```
scrape_jobs(site_name, search_term, location, ...)
        │
        ▼
1. Map site names to Scraper classes:
   "linkedin" → LinkedIn
   "indeed"   → Indeed
   "rozee"    → Rozee
        │
        ▼
2. Build ScraperInput from parameters
        │
        ▼
3. Create ThreadPoolExecutor
        │
        ▼
4. Submit each site's scraper as a CONCURRENT task
   ┌──────────┬──────────┬──────────┐
   │  Indeed   │ LinkedIn │  Rozee   │  ← run simultaneously
   │  Agent    │  Agent   │  Agent   │
   └──────────┴──────────┴──────────┘
        │
        ▼
5. Collect results as they complete (as_completed)
   - If any agent FAILS → log error, return empty JobResponse
   - Other agents continue unaffected (graceful degradation)
        │
        ▼
6. For each job from each agent:
   a. Convert to dict
   b. Add "site" field
   c. Format job_type (enum → string)
   d. Format location (Location object → display string)
   e. Extract/compute salary from compensation or description
   f. Extract skills (Naukri-specific)
   g. Create DataFrame row
        │
        ▼
7. Concatenate all DataFrames
8. Ensure all desired columns exist (add missing as None)
9. Reorder columns to standard order
10. Sort by site + date_posted
11. Return final Pandas DataFrame
```

### Why ThreadPoolExecutor?

Job boards are **I/O-bound** — they require HTTP requests that wait for responses. By using concurrent threads, we can scrape Indeed, LinkedIn, and Rozee simultaneously instead of sequentially. A request that would take 30 seconds serially takes ~10 seconds in parallel.

### Graceful Degradation

If Indeed is down but LinkedIn and Rozee work, the user still gets results from the working sites. One agent's failure never crashes the entire system.

---

## 5. Individual Scraper Agents

Each agent lives in its own directory under `jobspy/` and follows the same pattern:

| Agent | Directory | Platform | Method |
|-------|-----------|----------|--------|
| Indeed | `jobspy/indeed/` | Indeed.com | GraphQL API (mobile app UA) |
| LinkedIn | `jobspy/linkedin/` | LinkedIn.com | HTML scraping + optional description fetch |
| Rozee | `jobspy/rozee/` | Rozee.pk | HTML scraping |

**Currently active in SwipeSelect:** Indeed, LinkedIn, Rozee

Each agent extends the `Scraper` base class from `model.py`:
```python
class Scraper(ABC):
    def __init__(self, site, proxies=None, ca_cert=None, user_agent=None): ...
    
    @abstractmethod
    def scrape(self, scraper_input: ScraperInput) -> JobResponse: ...
```

---

## 6. Data Models & Schemas

**File:** `jobspy/model.py`

### Site Enum
```python
class Site(Enum):
    LINKEDIN = "linkedin"
    INDEED = "indeed"
    ROZEE = "rozee"
```

### Country Enum
Maps country names to subdomain codes for Indeed. Supports 60+ countries including Pakistan (`"pakistan", "pk"`). Each country entry contains:
- Country name(s)
- Indeed subdomain

### JobType Enum
Multilingual job type classification:
- `FULL_TIME` — includes translations in 20+ languages
- `PART_TIME`, `CONTRACT`, `TEMPORARY`, `INTERNSHIP`, `PER_DIEM`, `NIGHTS`, etc.

### JobPost Model (Internal)
The raw job post structure returned by individual scrapers before normalization. Contains site-specific fields like `company_logo`, `company_revenue`, `skills`, `experience_range`, etc.

### Location Model
```python
class Location(BaseModel):
    country: Country | str | None
    city: str | None
    state: str | None
    
    def display_location(self) -> str:
        # Returns "Lahore, Pakistan" from structured data
```

### Compensation Model
```python
class Compensation(BaseModel):
    interval: CompensationInterval  # yearly, monthly, hourly, etc.
    min_amount: float | None
    max_amount: float | None
    currency: str = "USD"
```

---

## 7. The Normalizer

**File:** `jobspy/normalizer.py`

The Normalizer transforms raw scraped DataFrame rows into the clean `JobPostSchema` that the backend expects. It's the **bridge between scraper internals and the API contract**.

### Key Methods:

**`to_iso_date(date_str)`** — Handles messy date formats:
- `"2 days ago"` → `"2026-05-10"`
- `"12h ago"` → `"2026-05-12"`
- `"04/28/2026"` → `"2026-04-28"`
- `"today"` → current date
- `None` → current date

**`clean_value(value, default)`** — Safely handles `None` and `NaN` values from Pandas

**`format_salary_range(row)`** — Creates human-readable salary strings:
- `"$80,000 - $120,000/yearly"`
- `"PKR50,000/monthly"`
- `None` if no salary data

**`extract_skills(row)`** — Extracts skills from various formats (list or comma-separated string)

**`to_job_post_schema(row)`** — The main transformation method:
```python
raw DataFrame row → JobPostSchema(
    title, company, description, location, url,
    salaryRange, jobType, source, postedAt, skills
)
```

---

## 8. Flask Backend API

**File:** `backend/app.py`

### `POST /api/search-jobs` — Main Search Endpoint

This is the primary endpoint that the Node.js backend calls:

```
1. Parse JSON request body
2. Validate: keywords required
3. Extract parameters with defaults:
   - keywords, location ("United States"), site_name (["indeed","linkedin"])
   - results_wanted (20), job_type, is_remote, hours_old (72)
   - offset (0), linkedin_fetch_description
4. Infer country from location (for Indeed subdomain)
5. Check in-memory cache (5-minute TTL)
6. Validate site names against AVAILABLE_SITES
7. Calculate scraper budget (how many results to ask each site for)
8. Call scrape_jobs() with parameters
9. Apply post-scrape filters (location, job_type, remote, freshness, keyword relevance)
10. If results < wanted and budget was limited → retry with full budget
11. Balance results across sites (round-robin selection)
12. Normalize via Normalizer.to_job_post_schema()
13. Build response with stats
14. Cache response
15. Return JSON
```

### Helper Functions:

**`infer_country(location)`** — Maps Pakistan city names to "Pakistan", defaults to "USA". This sets the correct Indeed subdomain.

**`normalize_site_names(site_name)`** — Ensures site names are lowercase list

**`calculate_scraper_result_budget(results_wanted, site_name)`** — Smart budget allocation:
- For ≤20 results: ask each site for the full amount
- For >20: ask each site for `ceil(wanted/sites) * 2` — a modest buffer for filtering

**`limit_results_balanced(jobs_df, results_wanted, site_name)`** — Round-robin selection from each site to ensure diversity (e.g., 7 Indeed + 7 LinkedIn + 6 Rozee for 20 results)

---

## 9. Filtering & Relevance Logic

The scraper applies intelligent post-scrape filtering:

### Discipline Mismatch Detection
```python
disciplines = {"software", "civil", "mechanical", "electrical", ...}

# If user searches "Software Engineer" but result is "Civil Engineer"
# → REJECTED (different discipline in title)
```

### Core Word Matching
```python
# Remove generic stop words: "engineer", "developer", "senior", etc.
# Check if remaining core words appear in title or description
# e.g., "software" must appear in title/desc for "Software Engineer" query
```

### Location Matching
- Handles variations: "USA" = "United States" = "US"
- "Pakistan" = any city containing "Pakistan"
- "Worldwide" or "Remote" matches everything

### Job Type Matching
- Normalizes hyphens and casing
- "Full-Time" matches "fulltime"

### Freshness Filter
- Compares `date_posted` against `hours_old` cutoff

---

## 10. Caching System

**In-memory dictionary cache** with 5-minute TTL:

```python
SEARCH_CACHE = {}
SEARCH_CACHE_TTL_SECONDS = 300

# Cache key = tuple of all search parameters
# Cache value = (timestamp, response_payload)
```

- Identical requests within 5 minutes return cached results instantly
- Prevents redundant scraping for repeated queries
- Cache is per-process (resets on restart)

---

## 11. Resilience & Anti-Bot Features

### User-Agent Rotation (`config.py`)
```python
DEFAULT_USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; ...) Chrome/121.0.0.0",
    "Mozilla/5.0 (Macintosh; ...) Chrome/120.0.0.0",
    "Mozilla/5.0 (X11; Linux ...) Chrome/119.0.0.0",
    "Mozilla/5.0 (iPhone; ...) Safari/604.1",
]

INDEED_USER_AGENTS = [
    "Mozilla/5.0 (iPhone; ...) Indeed App 193.1",  # Mobile app UA required
]
```

### TLS Client (`util.py`)
Uses `tls_client` library with `random_tls_extension_order=True` to mimic real browser TLS fingerprints. This bypasses bot detection that checks TLS handshake patterns.

### Rotating Proxy Support
```python
class RotatingProxySession:
    # Cycles through provided proxies for each request
    # Supports HTTP, HTTPS, and SOCKS5 proxies
```

### Retry Logic
```python
Retry(total=3, connect=3, status=3,
      status_forcelist=[500, 502, 503, 504, 429],
      backoff_factor=delay)
```

### Per-Site Custom Exceptions
Each site has its own exception class for clean error handling without affecting other agents.

---

## 12. API Endpoints Reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/health` | ❌ | Health check |
| POST | `/api/search-jobs` | 🔑 | Search for jobs (main endpoint) |
| GET | `/api/sites` | 🔑 | List available job sites |
| GET | `/api/job-types` | 🔑 | List available job types |

🔑 = Requires `x-scraper-api-key` header (if `SCRAPER_API_KEY` env var is set)

### Search Request Format
```json
{
  "keywords": "Software Engineer",
  "location": "Lahore, Pakistan",
  "site_name": ["indeed", "linkedin", "rozee"],
  "results_wanted": 20,
  "job_type": "fulltime",
  "is_remote": false,
  "hours_old": 168,
  "offset": 0,
  "linkedin_fetch_description": true
}
```

### Search Response Format
```json
{
  "success": true,
  "count": 20,
  "jobs": [
    {
      "title": "Full Stack Developer",
      "company": "TechCorp",
      "description": "We are looking for...",
      "location": "Lahore, Pakistan",
      "url": "https://indeed.com/...",
      "salaryRange": "PKR80,000 - PKR120,000/monthly",
      "jobType": "fulltime",
      "source": "indeed",
      "postedAt": "2026-05-10",
      "skills": ["React", "Node.js", "MongoDB"]
    }
  ],
  "stats": {
    "total_jobs": 20,
    "sites_scraped": ["indeed", "linkedin"],
    "sites_without_results": ["rozee"],
    "top_companies": {"TechCorp": 3, "Google": 2}
  }
}
```

---

## 13. Dependencies & Their Roles

| Package | Version | Purpose |
|---------|---------|---------|
| Flask | 3.0.3 | Web framework for the API |
| Flask-Cors | 4.0.1 | Cross-Origin Resource Sharing |
| gunicorn | 22.0.0 | Production WSGI server |
| requests | 2.31.0 | HTTP client for scraping |
| beautifulsoup4 | 4.12.3 | HTML parsing |
| pandas | 2.2.2 | DataFrame manipulation and job data processing |
| numpy | 1.26.4 | Numerical operations (salary parsing) |
| pydantic | 2.7.4 | Data validation and schema enforcement |
| tls-client | 1.0.1 | TLS fingerprint spoofing (anti-bot) |
| markdownify | 1.1.0 | HTML → Markdown conversion for descriptions |
| regex | 2024.5.15 | Advanced regex for salary/text extraction |

---

## 14. Design Decisions

| Decision | Rationale |
|----------|-----------|
| Python for scraping | Rich ecosystem: BeautifulSoup, requests, pandas, tls-client |
| Separate microservice | Independent scaling, language flexibility |
| ThreadPoolExecutor | I/O-bound tasks benefit from concurrency |
| Pandas DataFrame | Powerful data manipulation, filtering, sorting |
| Pydantic schemas | Strict type validation ensures clean API output |
| In-memory cache | Simple, fast, sufficient for single-process deployment |
| TLS client | Bypasses bot detection on Indeed/LinkedIn |
| Round-robin result balancing | Ensures diverse results from multiple sources |
| Post-scrape filtering | Scrapers return noisy data; filters ensure relevance |

---

## 15. How to Run

```bash
# 1. Navigate to the repo
cd Web-Scrapper

# 2. Create virtual environment
python -m venv jobspy_env
source jobspy_env/bin/activate  # or jobspy_env\Scripts\activate on Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set environment variables (optional)
export SCRAPER_API_KEY=your_secret_key
export PORT=5001

# 5. Run the server
cd backend
python app.py
```

---

## 16. Deployment

**Platform:** Render (Free Tier)

**`render.yaml` configuration:**
```yaml
services:
  - type: web
    name: swipe-select-scraper
    runtime: python
    plan: free
    buildCommand: pip install -r requirements.txt
    startCommand: gunicorn backend.app:app --bind 0.0.0.0:$PORT --workers 1 --threads 4 --timeout 120
    healthCheckPath: /api/health
    envVars:
      - key: PYTHON_VERSION
        value: 3.11.9
      - key: SCRAPER_API_KEY
        sync: false
```

- **1 worker, 4 threads** — optimal for I/O-bound scraping on free tier
- **120s timeout** — scraping can be slow, especially with LinkedIn description fetching
- **Health check** at `/api/health` — Render pings this to verify the service is alive

---

*This document covers every aspect of the Web-Scrapper. For how it connects to the Backend, see `INTEGRATION_EXPLAINED.md`.*
