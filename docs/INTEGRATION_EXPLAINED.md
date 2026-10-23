# How Backend & Scraper Work Together — Integration Deep-Dive

> This document explains exactly how the **Swipe-Select-Backend** (Node.js) and the **Web-Scrapper** (Python/Flask) communicate, what data flows between them, and how they combine to deliver job recommendations to the mobile app.

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Communication Protocol](#2-communication-protocol)
3. [End-to-End Request Flow](#3-end-to-end-request-flow)
4. [The Data Contract](#4-the-data-contract)
5. [Search Payload Construction](#5-search-payload-construction)
6. [Response Processing Pipeline](#6-response-processing-pipeline)
7. [Error Handling Across Services](#7-error-handling-across-services)
8. [Authentication Between Services](#8-authentication-between-services)
9. [Deployment Architecture](#9-deployment-architecture)
10. [Complete User Journey](#10-complete-user-journey)

---

## 1. System Architecture

```
┌─────────────────────┐         ┌─────────────────────┐         ┌─────────────────────┐
│                     │  HTTPS  │                     │  HTTPS  │                     │
│   Mobile App        │────────▶│   Backend           │────────▶│   Web-Scrapper      │
│   (React Native)    │◀────────│   (Node.js/Express) │◀────────│   (Python/Flask)    │
│                     │  JSON   │   Port 5000         │  JSON   │   Port 5001         │
│                     │  + JWT  │                     │         │                     │
└─────────────────────┘         │  ┌───────────────┐  │         │  ┌───────────────┐  │
                                │  │  MongoDB      │  │         │  │  Indeed Agent  │  │
                                │  │  Atlas        │  │         │  │  LinkedIn Agent│  │
                                │  │  (User, Job,  │  │         │  │  Rozee Agent   │  │
                                │  │   UserJob)    │  │         │  └───────────────┘  │
                                │  └───────────────┘  │         │                     │
                                └─────────────────────┘         └─────────────────────┘
```

### Key Rules
1. **Mobile app NEVER calls the scraper directly** — all requests go through the backend
2. Backend owns the user-facing API contract
3. Backend ↔ Scraper is an **internal** service-to-service call
4. JWT protects mobile → backend; API key protects backend → scraper

---

## 2. Communication Protocol

### Backend → Scraper

| Aspect | Detail |
|--------|--------|
| Protocol | HTTP POST |
| URL | `{SCRAPER_API_URL}/api/search-jobs` |
| Content-Type | `application/json` |
| Auth Header | `x-scraper-api-key: {SCRAPER_API_KEY}` |
| Timeout | 120 seconds |
| Abort | `AbortController` cancels if timeout exceeded |

### The Call (from `scraperClient.service.ts`):

```typescript
const response = await fetch(`${env.SCRAPER_API_URL}/api/search-jobs`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    ...(env.SCRAPER_API_KEY ? { 'x-scraper-api-key': env.SCRAPER_API_KEY } : {}),
  },
  body: JSON.stringify(payload),
  signal: controller.signal,  // 120s timeout
});
```

### The Receiver (from `backend/app.py`):

```python
@app.before_request
def require_api_key():
    if not SCRAPER_API_KEY or request.path == "/api/health":
        return None
    api_key = request.headers.get("x-scraper-api-key")
    if api_key != SCRAPER_API_KEY:
        return jsonify({"success": False, "error": "Unauthorized"}), 401
```

---

## 3. End-to-End Request Flow

Here is the **complete flow** when a user opens their job feed:

```
Step 1: Mobile App
────────────────────
User opens Dashboard → App calls GET /api/jobs/recommended
Headers: { Authorization: "Bearer <JWT>" }

        │
        ▼

Step 2: Backend Auth Middleware
────────────────────
protect() middleware:
  - Extract JWT from "Bearer <token>"
  - Verify with jwt.verify()
  - Load user from MongoDB
  - Attach to req.user

        │
        ▼

Step 3: Backend Controller (jobs.controller.ts)
────────────────────
getRecommendedJobs():
  - Check user.onboardingStep >= 13 (must complete onboarding)
  - Call getRecommendedJobsForUser(user)

        │
        ▼

Step 4: Backend Service — Check Cache (jobs.service.ts)
────────────────────
  - Query UserJob collection for active (queued/seen) jobs
  - Check if feed is fresh (lastRefreshedAt < 60 min ago)
  - If fresh and has jobs → RETURN cached jobs (skip scraping!)

        │  (if stale or empty)
        ▼

Step 5: Backend Service — Build Search Payload (jobSearch.service.ts)
────────────────────
buildSearchPayloadForUser(user):
  - Extract keywords from: preferences.jobTitles[0]
    OR profile.workExperience[0].title
    OR profile.skills.join(" ")
    OR "Software Engineer"
  - Extract location from: preferences.baseLocation
    OR preferences.targetCountries[0]
    OR "United States"
  - Normalize job type: "Full Time" → "fulltime"
  - Set is_remote = false (soft preference)
  - Set hours_old = 168 (7 days)
  - Set site_name = ["indeed", "linkedin", "rozee"]

Result:
{
  "keywords": "Full Stack Developer",
  "location": "Lahore, Pakistan",
  "site_name": ["indeed", "linkedin", "rozee"],
  "results_wanted": 20,
  "job_type": "fulltime",
  "is_remote": false,
  "hours_old": 168,
  "linkedin_fetch_description": true,
  "offset": 0
}

        │
        ▼

Step 6: Backend Service — Call Scraper (scraperClient.service.ts)
────────────────────
fetchJobsFromScraper(payload):
  - POST to http://scraper-url/api/search-jobs
  - 120s timeout with AbortController
  - Send x-scraper-api-key header

        │
        ▼

Step 7: Scraper — Receive & Validate (backend/app.py)
────────────────────
search_jobs() route:
  - Validate API key
  - Parse JSON body
  - Check cache (5-min TTL)
  - Validate site names
  - Infer country: "Lahore, Pakistan" → "Pakistan"

        │
        ▼

Step 8: Scraper — Concurrent Agent Dispatch (jobspy/__init__.py)
────────────────────
scrape_jobs():
  ThreadPoolExecutor launches 3 agents simultaneously:

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Indeed Agent  │  │LinkedIn Agent│  │ Rozee Agent  │
  │              │  │              │  │              │
  │ GraphQL API  │  │ HTML Scrape  │  │ HTML Scrape  │
  │ pk.indeed.com│  │ linkedin.com │  │ rozee.pk     │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                  │
         ▼                 ▼                  ▼
    [8 jobs]          [10 jobs]          [5 jobs]

  Agents that fail → return empty list (no crash)

        │
        ▼

Step 9: Scraper — Process Results (jobspy/__init__.py + backend/app.py)
────────────────────
  - Combine all agent results into single DataFrame (23 jobs)
  - Apply post-scrape filters:
    • Discipline mismatch check
    • Core keyword relevance
    • Location matching
    • Job type matching
    • Freshness check
  - Balance results across sites (round-robin)
  - Normalize via Normalizer.to_job_post_schema()
  - Cache response

        │
        ▼

Step 10: Scraper → Backend Response
────────────────────
{
  "success": true,
  "count": 20,
  "jobs": [
    {
      "title": "Full Stack Developer",
      "company": "TechCorp",
      "description": "We are looking for...",
      "location": "Lahore, Pakistan",
      "url": "https://indeed.com/viewjob?jk=abc123",
      "salaryRange": null,
      "jobType": "fulltime",
      "source": "indeed",
      "postedAt": "2026-05-10",
      "skills": ["React", "Node.js"]
    },
    ...
  ]
}

        │
        ▼

Step 11: Backend — Save & Deduplicate (jobs.service.ts)
────────────────────
saveScrapedJobsForUser(userId, scrapedJobs):
  For each scraped job:
    a. Normalize: trim, truncate description to 1500 chars
    b. Upsert into Job collection by sourceUrl (dedup!)
    c. Upsert UserJob link:
       - New: status="queued", recommendedAt=now
       - Existing: just update lastRefreshedAt

  Prune if > MAX_USER_JOBS (100) per user

        │
        ▼

Step 12: Backend → Mobile App Response
────────────────────
{
  "success": true,
  "data": {
    "jobs": [
      {
        "id": "683ae1f...",
        "title": "Full Stack Developer",
        "company": "TechCorp",
        "description": "We are looking for...",
        "location": "Lahore, Pakistan",
        "sourceUrl": "https://indeed.com/...",
        "salaryRange": null,
        "jobType": "fulltime",
        "source": "indeed",
        "postedAt": "2026-05-10T00:00:00.000Z",
        "skills": ["React", "Node.js"],
        "status": "queued"
      },
      ...
    ]
  },
  "message": null
}

        │
        ▼

Step 13: Mobile App — User Swipes
────────────────────
User swipes right → POST /api/jobs/:jobId/swipe { action: "like" }
Backend updates UserJob.status = "liked", swipedAt = now
```

---

## 4. The Data Contract

### What Backend Sends to Scraper
```typescript
interface ScraperSearchPayload {
  keywords: string;              // "Full Stack Developer"
  location: string;              // "Lahore, Pakistan"
  site_name: string[];           // ["indeed", "linkedin", "rozee"]
  results_wanted: number;        // 20
  job_type?: string;             // "fulltime"
  is_remote: boolean;            // false
  hours_old: number;             // 168
  linkedin_fetch_description?: boolean;  // true
  offset?: number;               // 0
}
```

### What Scraper Returns to Backend
```typescript
interface ScrapedJob {
  title: string;
  company: string;
  description: string;
  location: string;
  url: string;                   // Used as unique key for deduplication
  salaryRange?: string | null;
  jobType?: string | null;
  source: string;
  postedAt?: string | null;
  skills?: string[];
}
```

### What Backend Returns to Mobile App
```typescript
interface JobCard {
  id: string;                    // MongoDB ObjectId
  title: string;
  company: string;
  description: string;
  location: string;
  sourceUrl: string;
  salaryRange?: string | null;
  jobType?: string | null;
  source: string;
  postedAt?: Date | null;
  skills: string[];
  status: 'queued' | 'seen' | 'liked' | 'passed' | 'applied';
}
```

### Data Transformation Chain
```
Job Board HTML/API → Agent (raw) → DataFrame → Normalizer (JobPostSchema)
    → HTTP JSON → Backend (ScrapedJob) → MongoDB (IJob)
    → Populate + UserJob status → JobCard → Mobile App
```

---

## 5. Search Payload Construction

The backend intelligently builds search queries from user data:

```
User Document                    Search Payload
──────────────                   ──────────────
preferences.jobTitles[0]   ──→  keywords
  OR profile.workExperience[0].title
  OR profile.skills[0..2].join(" ")
  OR "Software Engineer"

preferences.baseLocation   ──→  location
  OR preferences.targetCountries[0]
  OR "United States"

preferences.jobTypes[0]    ──→  job_type (normalized)
  "Full Time" → "fulltime"

Always false                ──→  is_remote

["indeed","linkedin","rozee"] ──→  site_name

env.DEFAULT_JOB_RESULTS    ──→  results_wanted (20)

168 (7 days)                ──→  hours_old
```

---

## 6. Response Processing Pipeline

When the backend receives scraped jobs, it processes them through several stages:

```
Stage 1: Validation
  - Filter out jobs without URL or description

Stage 2: Normalization (normalizeScrapedJob)
  - Trim title, company, location
  - Truncate description to 1500 chars
  - Parse postedAt date string → Date object
  - Cap skills to 20 items
  - Default "Untitled job" / "Unknown Company" for missing fields

Stage 3: Deduplication (MongoDB upsert)
  - findOneAndUpdate by sourceUrl
  - If exists → update fields + extend expiresAt
  - If new → insert with all fields

Stage 4: User Association (UserJob upsert)
  - updateOne by userId + jobId
  - New: status="queued", recommendedAt=now
  - Existing: just refresh lastRefreshedAt + expiresAt

Stage 5: Pruning
  - If user has > 100 UserJobs → delete oldest
```

---

## 7. Error Handling Across Services

### Scraper Failures
```
Scenario: Scraper service is down or returns error
    │
    ├── Not force-refresh + has cached jobs
    │   └── Return cached jobs (user sees stale but working data)
    │
    ├── Not force-refresh + no cached jobs
    │   └── Throw ScraperClientError → 503 to mobile app
    │
    └── Force-refresh
        └── Throw ScraperClientError → 503 to mobile app
            "Could not fetch jobs right now. Please try again."
```

### Individual Agent Failures
```
Scenario: Indeed is blocked, LinkedIn works, Rozee works
    │
    └── ThreadPoolExecutor catches the exception
        └── Indeed returns empty JobResponse
        └── LinkedIn and Rozee results are still used
        └── User gets partial but usable results
```

### Timeout Handling
```
Backend → Scraper: AbortController with 120s timeout
    │
    └── If timeout → ScraperClientError thrown
        └── Same graceful degradation as above
```

---

## 8. Authentication Between Services

```
Mobile App                    Backend                      Scraper
──────────                    ───────                      ───────
                              
JWT in Authorization          Verifies JWT                 
header ──────────────────────▶with JWT_SECRET              
                              Loads user from DB           
                                                           
                              x-scraper-api-key   ────────▶Checks against
                              header                       SCRAPER_API_KEY
                                                           env variable
```

- **Mobile → Backend:** JWT token (user identity)
- **Backend → Scraper:** API key (service identity)
- **Scraper health check:** No auth required (`/api/health` is exempted)

---

## 9. Deployment Architecture

Both services are deployed on **Render**:

```
┌─────────────────────────────────────────────────┐
│                   Render Cloud                   │
│                                                   │
│  ┌─────────────────────┐  ┌─────────────────────┐│
│  │ swipe-select-backend │  │ swipe-select-scraper ││
│  │                     │  │                     ││
│  │ Runtime: Node.js    │  │ Runtime: Python     ││
│  │ Build: npm run build│  │ Build: pip install  ││
│  │ Start: npm start    │  │ Start: gunicorn     ││
│  │                     │  │                     ││
│  │ SCRAPER_API_URL ────┼──┼→ Internal URL       ││
│  │ SCRAPER_API_KEY ────┼──┼→ Shared secret      ││
│  └─────────────────────┘  └─────────────────────┘│
│                                                   │
│  ┌─────────────────────┐                         │
│  │ MongoDB Atlas        │ (external)              │
│  │ (cloud database)    │                         │
│  └─────────────────────┘                         │
└─────────────────────────────────────────────────┘
```

**Environment Variables Connection:**
- Backend's `SCRAPER_API_URL` = Scraper's Render URL (e.g., `https://swipe-select-scraper.onrender.com`)
- Backend's `SCRAPER_API_KEY` = Scraper's `SCRAPER_API_KEY` (same shared secret)

---

## 10. Complete User Journey

Here's the **full lifecycle** from registration to swiping:

```
1. REGISTRATION
   Mobile → POST /api/auth/register → Backend creates User (onboardingStep=0)
   
2. RESUME UPLOAD (Onboarding Step 1)
   Mobile → POST /api/onboarding/resume-extract (PDF file)
   Backend → pdf-parse extracts text → Gemini AI parses → saves profile
   onboardingStep = 1

3. ONBOARDING (Steps 2-13)
   Mobile → POST /api/onboarding/preferences (multiple calls)
   Each screen saves: jobTitles, location, gender, country, workMode, etc.
   onboardingStep increments to 13

4. FIRST JOB LOAD
   Mobile → GET /api/jobs/recommended
   Backend checks: onboardingStep=13 ✓, no cached jobs
   Backend → builds search from preferences → calls Scraper
   Scraper → launches Indeed+LinkedIn+Rozee agents concurrently
   Scraper → filters, normalizes, returns 20 jobs
   Backend → saves to MongoDB, creates UserJob links, returns JobCards

5. SWIPING
   Mobile → POST /api/jobs/:id/swipe { action: "like" }
   Backend → updates UserJob status to "liked"
   
   Mobile → POST /api/jobs/:id/swipe { action: "pass" }
   Backend → updates UserJob status to "passed"

6. VIEW MATCHES
   Mobile → GET /api/jobs/matches
   Backend → returns all jobs where status = "liked" or "applied"

7. REFRESH (when user pulls-to-refresh or feed runs out)
   Mobile → POST /api/jobs/refresh
   Backend → force-fetches new jobs from Scraper
   Scraper → runs fresh search, returns new results
   Backend → saves, deduplicates, returns updated feed

8. AUTO-CLEANUP (background)
   MongoDB TTL indexes automatically delete:
   - Jobs older than 14 days (expiresAt field)
   - UserJobs older than 14 days
   No cron jobs needed!
```

---

## Summary Table

| Aspect | Backend (Node.js) | Scraper (Python) |
|--------|-------------------|------------------|
| **Port** | 5000 | 5001 |
| **Framework** | Express 5 | Flask |
| **Language** | TypeScript | Python |
| **Database** | MongoDB Atlas | None (stateless) |
| **Auth (incoming)** | JWT tokens | API key header |
| **Caching** | MongoDB (TTL indexes) | In-memory (5 min) |
| **Concurrency** | Single-threaded (Node event loop) | ThreadPoolExecutor |
| **Deployment** | Render (Node.js) | Render (Python + Gunicorn) |
| **Primary Role** | Business logic, data persistence | Job aggregation from multiple sources |

---

*For individual component details, see `BACKEND_EXPLAINED.md` and `SCRAPER_EXPLAINED.md`.*
