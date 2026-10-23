# Swipe-Select Backend — Complete Technical Deep-Dive

> **Repo:** `Swipe-Select-Backend`  
> **Tech Stack:** Node.js · Express 5 · TypeScript · MongoDB (Mongoose) · JWT · Zod · Gemini AI  
> **Port:** `5000`  
> **Purpose:** Central REST API that powers the SwipeSelect mobile app — handles authentication, user onboarding, AI resume parsing, job scraping orchestration, and the swipe-based job matching system.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [How the Server Starts](#2-how-the-server-starts)
3. [Configuration & Environment](#3-configuration--environment)
4. [Database Connection](#4-database-connection)
5. [Data Models (MongoDB Schemas)](#5-data-models-mongodb-schemas)
6. [Authentication System](#6-authentication-system)
7. [Middleware Layer](#7-middleware-layer)
8. [Onboarding System & AI Resume Parsing](#8-onboarding-system--ai-resume-parsing)
9. [Job Discovery & Recommendation Engine](#9-job-discovery--recommendation-engine)
10. [API Endpoints Reference](#10-api-endpoints-reference)
11. [API Response Contract](#11-api-response-contract)
12. [Swagger / OpenAPI Documentation](#12-swagger--openapi-documentation)
13. [Security Measures](#13-security-measures)
14. [Design Decisions & Trade-Offs](#14-design-decisions--trade-offs)
15. [How to Run](#15-how-to-run)

---

## 1. Project Structure

```
Swipe-Select-Backend/
├── src/
│   ├── server.ts                    # Entry point — connects DB, starts Express
│   ├── app.ts                       # Express app setup (middlewares, routes, error handler)
│   ├── config/
│   │   ├── env.ts                   # Centralized environment variable configuration
│   │   ├── db.ts                    # MongoDB connection via Mongoose
│   │   └── swagger.ts               # Swagger/OpenAPI spec definition
│   ├── models/
│   │   ├── user.model.ts            # User schema (profile, preferences, auth)
│   │   ├── job.model.ts             # Cached job listing schema
│   │   └── userJob.model.ts         # User-Job link (swipe state tracking)
│   ├── controllers/
│   │   ├── auth.controller.ts       # Register, Login, Google OAuth, Profile, Logout
│   │   ├── onboarding.controller.ts # Resume extraction + preference saving
│   │   └── jobs.controller.ts       # Recommended jobs, refresh, swipe, matches
│   ├── services/
│   │   ├── resume.service.ts        # PDF text extraction + Gemini AI parsing
│   │   ├── scraperClient.service.ts # HTTP client to call the Python scraper
│   │   ├── jobSearch.service.ts     # Builds search payload from user preferences
│   │   └── jobs.service.ts          # Core job recommendation logic (fetch, save, swipe)
│   ├── middlewares/
│   │   ├── auth.middleware.ts       # JWT verification + user attachment to request
│   │   └── errorHandler.ts          # Global Express error handler
│   ├── routes/
│   │   ├── auth.routes.ts           # /api/auth/* routes with Swagger JSDoc
│   │   ├── onboarding.routes.ts     # /api/onboarding/* routes
│   │   ├── jobs.routes.ts           # /api/jobs/* routes
│   │   └── health.routes.ts         # /api/health route
│   └── utils/
│       └── jwt.ts                   # JWT sign + verify helpers
├── package.json
├── tsconfig.json
└── .env.example
```

**Architecture Pattern:** `Route → Controller → Service → Model`

This is a layered architecture:
- **Routes** define HTTP method + path + middleware chain
- **Controllers** handle request parsing, validation (Zod), and response formatting
- **Services** contain all business logic (decoupled from HTTP layer)
- **Models** define MongoDB schemas and data access

---

## 2. How the Server Starts

### `server.ts` (Entry Point)
```
1. Import the Express app from app.ts
2. Import connectDB from config/db.ts
3. Import env from config/env.ts
4. Call connectDB() — connects to MongoDB Atlas
5. Call app.listen(env.PORT) — starts HTTP server on port 5000
```

### `app.ts` (Express Setup)
```
1. Create Express app instance
2. Apply security middleware: helmet() — sets security HTTP headers
3. Apply cors() — enables Cross-Origin Resource Sharing
4. Apply body parsers: express.json() + express.urlencoded()
5. Mount route groups:
   - /api/health   → health.routes.ts
   - /api/auth     → auth.routes.ts
   - /api/onboarding → onboarding.routes.ts
   - /api/jobs     → jobs.routes.ts
6. Mount Swagger UI at /api-docs
7. Add 404 fallback handler
8. Add global error handler
```

---

## 3. Configuration & Environment

**File:** `src/config/env.ts`

All environment variables are centralized in a single `env` object with sensible defaults:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `5000` | Server port |
| `MONGO_URI` | `mongodb://localhost:27017/swipe-select` | MongoDB connection string |
| `JWT_SECRET` | `fallback_secret_key_change_in_prod` | Secret key for signing JWTs |
| `JWT_EXPIRES_IN` | `7d` | Token expiration time |
| `NODE_ENV` | `development` | Environment mode |
| `GOOGLE_CLIENT_IDS` | `[]` | Comma-separated Google OAuth client IDs |
| `GEMINI_API_KEY` | `''` | Google Gemini AI API key for resume parsing |
| `SCRAPER_API_URL` | `http://localhost:5001` | URL of the Python scraper service |
| `SCRAPER_API_KEY` | `''` | API key for authenticating with the scraper |
| `JOB_REFRESH_TTL_MINUTES` | `60` | How long before jobs are considered stale |
| `DEFAULT_JOB_RESULTS` | `20` | Number of jobs to fetch per request |
| `JOB_RETENTION_DAYS` | `14` | Auto-delete jobs older than this |
| `MAX_USER_JOBS` | `100` | Max UserJob records per user (prevents bloat) |
| `JOB_DESCRIPTION_MAX_LENGTH` | `1500` | Truncate long descriptions to save storage |

---

## 4. Database Connection

**File:** `src/config/db.ts`

- Uses Mongoose to connect to MongoDB Atlas
- On success: logs the connected host
- On failure: logs the error and **kills the process** (`process.exit(1)`) — this is intentional so the deployment platform (Render) can detect the crash and retry

---

## 5. Data Models (MongoDB Schemas)

### 5.1 User Model (`user.model.ts`)

The User document is the **central entity** — it stores everything about a user in one document.

```typescript
interface IUser {
  name: string;               // Display name
  email: string;              // Unique, lowercase, trimmed
  password?: string;          // Hashed with bcrypt (optional for Google users)
  googleId?: string;          // For Google OAuth (sparse unique index)
  onboardingStep: number;     // 0-13, tracks progress through onboarding
  profile: {                  // Populated by AI resume parsing
    firstName, lastName, gender, phone,
    education[],               // { school, degree, date }
    workExperience[],          // { title, company, location, startDate, endDate }
    projects[],                // { name, website, date, description }
    skills[],                  // ["JavaScript", "React", ...]
    certifications[],          // { name, link }
    interests[]
  };
  preferences: {              // Set during onboarding
    jobTitles[],              // What jobs they're looking for
    targetCountries[],        // Where they want to work
    baseLocation,             // Their current city
    workLocations[],          // Remote, Hybrid, On-site
    workMode[],               // Full-time, Part-time, etc.
    jobTypes[],               // Permanent, Contract, etc.
    experienceLevel[]         // Junior, Mid, Senior
  };
}
```

**Key Features:**
- **Password hashing:** Pre-save hook uses `bcrypt.genSalt(10)` + `bcrypt.hash()` — password is automatically hashed before saving to DB
- **Password verification:** `matchPassword()` method uses `bcrypt.compare()` for login
- **Embedded sub-documents:** Profile and preferences are embedded (not separate collections) because they're always read together with the user and are bounded in size

### 5.2 Job Model (`job.model.ts`)

Stores normalized scraped job listings. These are **shared across users** — the same job can be recommended to multiple users.

```typescript
interface IJob {
  title: string;
  company: string;
  description: string;       // Truncated to 1500 chars
  location: string;
  sourceUrl: string;          // UNIQUE — used for deduplication
  salaryRange?: string;
  jobType?: string;
  source: string;             // "indeed", "linkedin", "rozee"
  postedAt?: Date;
  skills: string[];           // Up to 20 skills
  expiresAt: Date;            // TTL — auto-deleted after 14 days
}
```

**Indexes:**
1. `{ source: 1, postedAt: -1 }` — fast queries by source sorted by date
2. `{ title: 'text', company: 'text', description: 'text', skills: 'text' }` — full-text search
3. `{ expiresAt: 1 }` with `expireAfterSeconds: 0` — **MongoDB TTL index** that automatically deletes expired documents

### 5.3 UserJob Model (`userJob.model.ts`)

This is the **junction table** between Users and Jobs. It tracks each user's relationship with each job (their swipe state).

```typescript
type UserJobStatus = 'queued' | 'seen' | 'liked' | 'passed' | 'applied';

interface IUserJob {
  userId: ObjectId;           // Reference to User
  jobId: ObjectId;            // Reference to Job
  status: UserJobStatus;      // Current swipe state
  recommendedAt: Date;        // When this job was recommended
  lastRefreshedAt: Date;      // Last time this recommendation was refreshed
  swipedAt?: Date;            // When the user swiped (null if not swiped)
  expiresAt: Date;            // TTL — auto-delete after 14 days
}
```

**Indexes:**
1. `{ userId: 1, jobId: 1 }` — **compound unique index** prevents duplicate recommendations
2. `{ userId: 1, status: 1, recommendedAt: -1 }` — fast "get my unswiped jobs" query
3. `{ expiresAt: 1 }` — TTL auto-cleanup

**Why separate from User?** Because user-job relationships are **unbounded** — a user could swipe on hundreds of jobs. Embedding this in the User document would cause it to grow indefinitely and exceed MongoDB's 16MB document limit.

---

## 6. Authentication System

### 6.1 JWT Utilities (`utils/jwt.ts`)

- **`generateToken(userId)`** — Signs a JWT with `{ id: userId }` as payload, using `env.JWT_SECRET`, expiring in `7d`
- **`verifyToken(token)`** — Verifies and decodes a JWT token

### 6.2 Auth Controller (`controllers/auth.controller.ts`)

#### Registration (`POST /api/auth/register`)
1. Validates input with Zod: `name` (min 2 chars), `email` (valid email), `password` (min 6 chars)
2. Checks if email already exists → 400 error if so
3. Creates user via `User.create()` — password is auto-hashed by the pre-save hook
4. Returns user data + JWT token

#### Login (`POST /api/auth/login`)
1. Validates with Zod: `email` + `password`
2. Finds user by email
3. Calls `user.matchPassword(password)` — bcrypt comparison
4. Returns user data + JWT token (or 401 for invalid credentials)

#### Google OAuth (`POST /api/auth/google`)
1. Receives Google `idToken` from the mobile app
2. Verifies token with Google's `OAuth2Client.verifyIdToken()` against allowed client IDs
3. If user exists with that email → links their `googleId` (if not already linked)
4. If new user → creates account without a password
5. Returns user data + JWT token

#### Profile (`GET /api/auth/profile`)
- Protected by JWT middleware
- Retrieves full user document (excluding password)

#### Logout (`POST /api/auth/logout`)
- Returns success response — actual token removal happens on the client
- Exists for future token blacklisting or audit logging

---

## 7. Middleware Layer

### 7.1 Auth Middleware (`middlewares/auth.middleware.ts`)

Protects routes that require authentication:

```
1. Check for "Bearer <token>" in Authorization header
2. Extract the token
3. Verify with jwt.verify()
4. Find user in DB by decoded ID (excluding password)
5. Attach user to req.user
6. Call next() — or return 401 if anything fails
```

### 7.2 Error Handler (`middlewares/errorHandler.ts`)

Global Express error handler (4-argument middleware):
- Returns `{ success: false, message, stack? }`
- Stack trace only included in `development` mode
- Defaults to 500 status if no `statusCode` is set on the error

---

## 8. Onboarding System & AI Resume Parsing

### 8.1 Resume Extraction Flow

**Endpoint:** `POST /api/onboarding/resume-extract`  
**File upload:** Uses `multer` with in-memory storage, 5MB limit, field name `resume`

The flow:

```
1. User uploads PDF resume from mobile app
       │
       ▼
2. Multer stores file in memory (req.file.buffer)
       │
       ▼
3. extractTextFromPDF(buffer)
   - Uses @cedrugs/pdf-parse to extract raw text from PDF
   - Returns plain text string
       │
       ▼
4. parseResumeWithGemini(text)
   - Sends text to Google Gemini AI (gemini-flash-latest model)
   - Uses STRUCTURED OUTPUT with a JSON schema:
     {
       firstName, lastName, gender, phone,
       education[], workExperience[], projects[],
       skills[], certifications[], interests[]
     }
   - Gemini returns a JSON object matching the schema
       │
       ▼
5. Atomically updates user document:
   - Sets profile = parsed data
   - Sets onboardingStep = 1
       │
       ▼
6. Returns parsed profile to mobile app
```

**Why Gemini with Structured Output?** Instead of relying on regex or manual parsing (which breaks on every resume format), we use Gemini AI with a **forced JSON schema** (`responseMimeType: 'application/json'` + `responseSchema`). This guarantees the output is always valid JSON matching our expected structure. The AI handles the intelligence of extracting names, skills, education, etc. from unstructured resume text.

### 8.2 Preferences Saving

**Endpoint:** `POST /api/onboarding/preferences`

- Validates with Zod schema (all fields optional)
- Accepts: `jobTitles`, `targetCountries`, `baseLocation`, `workLocations`, `workMode`, `jobTypes`, `experienceLevel`, `onboardingStep`, `profile`
- Uses MongoDB's `$set` operator for **atomic, partial updates** — only updates fields that were sent
- For nested profile fields: uses dot notation (`profile.gender`, `profile.firstName`) to avoid overwriting the entire profile
- Returns updated preferences, profile, and onboarding step

---

## 9. Job Discovery & Recommendation Engine

This is the most complex part of the backend. Here's how the entire job recommendation pipeline works:

### 9.1 Building the Search Payload (`jobSearch.service.ts`)

When a user requests jobs, we need to convert their preferences into a search query for the scraper:

```
User Preferences → Search Payload Builder → Scraper API Request
```

**Keyword Priority:**
1. First preferred job title (e.g., "Full Stack Developer")
2. First work experience title (e.g., "Junior MERN Stack Developer")
3. First 3 skills joined (e.g., "JavaScript TypeScript React")
4. Fallback: "Software Engineer"

**Location Priority:**
1. Base location (e.g., "Lahore, Pakistan")
2. First target country (e.g., "Pakistan")
3. First work location
4. Fallback: "United States"

**Job Type Normalization:** Maps user preferences like "Full Time" → "fulltime" (what the scraper expects)

**Important Design Decision:** `is_remote` is always set to `false` in the scraper request. Why? Pakistan job boards don't reliably label jobs as remote, so using it as a hard filter would eliminate valid results. Remote preference is treated as a soft filter for ranking/UI only.

### 9.2 Scraper Client (`scraperClient.service.ts`)

HTTP client that calls the Python scraper service:

- **URL:** `env.SCRAPER_API_URL` + `/api/search-jobs`
- **Timeout:** 120 seconds (configurable via `SCRAPER_TIMEOUT_MS`)
- **Auth:** Sends `x-scraper-api-key` header if configured
- **AbortController:** Uses `AbortController` to timeout the request
- **Error handling:** Wraps all errors in `ScraperClientError` class for clean error propagation

### 9.3 Core Recommendation Logic (`jobs.service.ts`)

This is the heart of the system. Here's the step-by-step flow:

```
GET /api/jobs/recommended
        │
        ▼
1. Check onboardingStep >= 13 (user must complete onboarding)
        │
        ▼
2. Get existing active (queued/seen) UserJobs for this user
        │
        ▼
3. Check if feed is fresh (last refresh < TTL_MINUTES ago)
   ├── YES and not force-refresh → Return existing jobs
   │
   └── NO or force-refresh
        │
        ▼
4. Build search payload from user preferences
        │
        ▼
5. Call Python scraper (fetchJobsFromScraper)
        │
        ▼
6. For each scraped job:
   a. Normalize: trim strings, truncate description, parse dates
   b. Upsert into Job collection (deduplicate by sourceUrl)
   c. Upsert UserJob link (create if new, update lastRefreshedAt)
        │
        ▼
7. Prune excess UserJobs (keep only MAX_USER_JOBS per user)
        │
        ▼
8. Return fresh active UserJobs with populated Job data
```

**Key Concepts:**

- **TTL-based freshness:** Jobs are only re-fetched if the last refresh was more than 60 minutes ago (configurable). This prevents hammering the scraper service.
- **Graceful degradation:** If the scraper fails but we have cached jobs, we return the cached ones. Only throw if forced refresh and scraper fails.
- **Deduplication:** Jobs are deduped by `sourceUrl` — if the same Indeed listing is scraped twice, it updates the existing record instead of creating a duplicate.
- **Pruning:** Old UserJob records are automatically deleted to prevent unbounded growth (max 100 per user).

### 9.4 Swipe Logic

```
POST /api/jobs/:jobId/swipe   { action: "like" | "pass" | "apply" }
        │
        ▼
1. Validate jobId is a valid ObjectId
2. Map action → status: like→"liked", pass→"passed", apply→"applied"
3. Find UserJob by userId + jobId
4. Update status and set swipedAt timestamp
5. Return updated swipe state
```

### 9.5 Matched Jobs

```
GET /api/jobs/matches
        │
        ▼
1. Find all UserJobs where status is "liked" or "applied"
2. Sort by swipedAt (most recent first)
3. Populate the Job data
4. Return as JobCard array
```

---

## 10. API Endpoints Reference

### Health
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/health` | ❌ | Server health check |

### Authentication
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/register` | ❌ | Register with email/password |
| POST | `/api/auth/login` | ❌ | Login with email/password |
| POST | `/api/auth/google` | ❌ | Login/Register with Google OAuth |
| GET | `/api/auth/profile` | ✅ | Get current user profile |
| POST | `/api/auth/logout` | ✅ | Logout (signal to server) |

### Onboarding
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/onboarding/resume-extract` | ✅ | Upload PDF + AI-parse resume |
| POST | `/api/onboarding/preferences` | ✅ | Save user preferences/profile |

### Jobs
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/jobs/recommended` | ✅ | Get swipeable job feed |
| POST | `/api/jobs/refresh` | ✅ | Force-refresh job feed |
| POST | `/api/jobs/:jobId/swipe` | ✅ | Swipe on a job (like/pass/apply) |
| GET | `/api/jobs/matches` | ✅ | Get liked/applied jobs |

---

## 11. API Response Contract

Every response follows a consistent format:

```json
{
  "success": true | false,
  "data": { ... } | null,
  "message": "string" | null
}
```

- `success: true` + `data` = successful operation
- `success: false` + `message` = error with description

---

## 12. Swagger / OpenAPI Documentation

- Auto-generated from JSDoc comments in route files
- Available at `GET /api-docs`
- Uses OpenAPI 3.0 spec
- Defines reusable schemas: `User`, `Profile`, `Preferences`, `AuthResponse`, `ErrorResponse`
- Supports `bearerAuth` security scheme for protected endpoints

---

## 13. Security Measures

1. **Helmet:** Sets various HTTP headers to prevent common attacks (XSS, clickjacking, etc.)
2. **CORS:** Enabled for all origins (configurable for production)
3. **bcrypt:** Passwords hashed with salt factor 10
4. **JWT:** Stateless authentication with 7-day expiration
5. **Zod Validation:** All request bodies are validated before processing — prevents malformed data
6. **Multer File Limits:** Resume uploads capped at 5MB
7. **Sparse Unique Index:** `googleId` is unique but allows `null` (users without Google auth)
8. **No Password in Responses:** User queries always `.select('-password')`

---

## 14. Design Decisions & Trade-Offs

| Decision | Rationale |
|----------|-----------|
| Profile/Preferences embedded in User | Bounded data that's always read together — single query, no joins |
| Jobs and UserJobs in separate collections | Unbounded data — avoids document bloat, enables TTL indexes |
| Remote filter = false | Pakistan job boards don't reliably label remote jobs |
| Description capped at 1500 chars | Prevents storage bloat without losing useful info |
| TTL indexes (14 days) | Auto-cleanup of stale data, zero maintenance |
| Scraper as separate service | Independent scaling, language flexibility (Python for scraping) |
| JWT stateless auth | No session storage needed, scales horizontally |
| Gemini structured output | Guarantees valid JSON from AI — no regex resume parsing |

---

## 15. How to Run

### Prerequisites
- Node.js 18+
- MongoDB Atlas account (or local MongoDB)
- Google Gemini API key
- Python scraper service running on port 5001

### Steps
```bash
# 1. Install dependencies
cd Swipe-Select-Backend
npm install

# 2. Create .env file from template
cp .env.example .env
# Edit .env with your actual values

# 3. Run in development mode
npm run dev

# 4. Build for production
npm run build

# 5. Start production server
npm start
```

### Deployment (Render)
- **Build Command:** `npm run build`
- **Start Command:** `npm start`
- **Environment Variables:** Set all from `.env.example` in Render dashboard

---

*This document covers every aspect of the Swipe-Select Backend. For questions about how it connects to the Web Scrapper, see `INTEGRATION_EXPLAINED.md`.*
