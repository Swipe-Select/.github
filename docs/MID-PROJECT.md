# SwipeSelect - Project Overview

## What is SwipeSelect?

SwipeSelect is a mobile job discovery application where users swipe through job listings to find opportunities that match their skills and preferences. Think of it as "Tinder for jobs" — users swipe right on jobs they're interested in and left on those they pass.

The system consists of **three interconnected repositories** working together to provide a seamless job search experience.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SwipeSelect System                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────┐       ┌─────────────────────────┐       │
│  │   Mobile App (React      │       │   Backend (Node.js/     │       │
│  │   Native/Expo)           │       │   Express/TypeScript)   │       │
│  │   Port: N/A (dev)        │──────>│   Port: 5000            │       │
│  │                         │  JWT  │                         │       │
│  └─────────────────────────┘       │  ┌───────────────────┐  │       │
│                                     │  │  Python Scraper  │  │       │
│                                     │  │  (Flask/JobSpy)  │  │       │
│                                     │  │  Port: 5001      │  │       │
│                                     │  └───────────────────┘  │       │
│                                     └─────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Descriptions

### 1. Swipe-Select-MobileApp (React Native / Expo)

**Purpose:** The frontend mobile application that users interact with.

**Tech Stack:**
- React Native with Expo
- TypeScript
- Expo Font (Poppins + Outfit typography)
- JWT-based authentication

**Key Features:**
- **Authentication Flow:** Ready → Sign Up / Login → Onboarding
- **Onboarding Flow:** 13 screens guiding users through resume upload and preferences
- **Job Dashboard:** Swipe-ready job cards
- **Profile Management:** User profile and preferences

**Onboarding Screens (in order):**
1. Resume Upload
2. Ready to Swipe
3. Personal Information
4. Job Type Selection
5. Location Setting
6. Gender Selection
7. Country Selection
8. Notifications
9. Resume Builder
10. Target Work Location
11. Work Preference
12. Employment Model
13. Experience Level

**Key Files:**
- `src/app/AppRoot.tsx` - Main entry point with auth gate
- `src/features/auth/` - Authentication screens
- `src/features/onboarding/` - 13 onboarding screens
- `src/features/onboarding/screens/ResumeUploadScreen.tsx` - Resume parsing trigger

**App Identifiers:**
- iOS Bundle ID: `com.swipeselect.mobileapp`
- Android Package: `com.swipeselect.mobileapp`

---

### 2. Swipe-Select-Backend (Node.js / Express)

**Purpose:** Central API that handles all business logic, authentication, and orchestrates the job discovery flow. The mobile app communicates ONLY with this backend.

**Tech Stack:**
- Node.js + Express + TypeScript
- MongoDB (Atlas) with Mongoose
- JWT authentication
- Zod validation

**Architecture Pattern:** Route → Controller → Service → Model

**Core Features:**

#### Authentication
- Email/password registration and login
- Google OAuth integration
- JWT token-based session management
- Profile retrieval and logout

#### Onboarding
- Resume upload (PDF) with text extraction via `cedrugs/pdf-parse`
- Resume parsing using Gemini AI
- Preferences saving (job titles, locations, work mode, etc.)
- Onboarding step tracking (0-13)

#### Job Discovery
- Fetches jobs from the Python scraper service
- Normalizes and deduplicates scraped jobs
- Manages user swipe state (like/pass/apply)
- Returns swipe-ready job cards

**API Response Format:**
```json
{
  "success": true,
  "data": {},
  "message": null
}
```

**Key Endpoints:**

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/auth/register` | POST | Email/password registration |
| `/api/auth/login` | POST | Email/password login |
| `/api/auth/google` | POST | Google OAuth login |
| `/api/auth/profile` | GET | Get current user profile |
| `/api/auth/logout` | POST | Logout |
| `/api/onboarding/resume-extract` | POST | Upload and parse resume |
| `/api/onboarding/preferences` | POST | Save user preferences |
| `/api/jobs/recommended` | GET | Get swipeable job feed |
| `/api/jobs/refresh` | POST | Force fresh job fetch |
| `/api/jobs/:jobId/swipe` | POST | Swipe on a job |

**Data Models:**

- **User:** Embedded profile and preferences (bounded data)
- **Job:** Normalized scraped job (cached, TTL-based)
- **UserJob:** Per-user swipe state linking user to job

---

### 3. Web-Scrapper (Python / Flask)

**Purpose:**独立 job scraping service that aggregates listings from multiple job boards.

**Tech Stack:**
- Python
- Flask
- JobSpy library

**How It Works:**

The system uses an **Agent-based architecture**:

1. Flask backend receives search requests
2. JobSpy dispatches specialized **Scraper Agents** for each platform
3. Agents run concurrently via ThreadPoolExecutor
4. Results are harmonized and normalized
5. Consolidated DataFrame returned to caller

**Supported Job Boards (Agents):**
| Agent | Platform |
|-------|----------|
| Indeed Agent | Indeed |
| LinkedIn Agent | LinkedIn |
| Glassdoor Agent | Glassdoor |
| Google Agent | Google Jobs |
| ZipRecruiter Agent | ZipRecruiter |
| Naukri/BDJobs/Bayt | International (Asia/Middle East) |

**Scraper Contract (Backend → Scraper):**
```json
{
  "keywords": "Software Engineer",
  "location": "Pakistan",
  "site_name": ["indeed", "linkedin", "rozee"],
  "results_wanted": 20,
  "job_type": "fulltime",
  "is_remote": false,
  "hours_old": 168,
  "offset": 0
}
```

**Resilience Features:**
- User-Agent rotation
- Proxy support
- Graceful degradation (one agent failure doesn't break others)
- Schema validation with Pydantic

---

## How They Connect

### Communication Flow

```
1. User opens app
   │
   ▼
2. Mobile App ──▶ Backend: POST /api/auth/login
   │              (with credentials)
   │
   ▼
3. Backend validates, returns JWT + user data
   │
   ▼
4. Mobile App ──▶ Backend: GET /api/auth/profile
   │              (with JWT)
   │
   ▼
5. User completes onboarding (resume + preferences saved)
   │
   ▼
6. Mobile App (Dashboard) ──▶ Backend: GET /api/jobs/recommended
   │                           (with JWT)
   │
   ▼
7. Backend reads user preferences, builds scraper request
   │
   ▼
8. Backend ──▶ Scraper: POST /api/search-jobs
   │            (internal, no JWT)
   │
   ▼
9. Scraper queries Indeed, LinkedIn, Rozee, etc.
   │
   ▼
10. Scraper ──▶ Backend: Returns normalized jobs
    │
    ▼
11. Backend deduplicates, saves to MongoDB, returns to mobile
    │
    ▼
12. User swipes on job cards
    │
    ▼
13. Mobile App ──▶ Backend: POST /api/jobs/:jobId/swipe
    │              (action: like/pass/apply)
```

### Key Rules

1. **Mobile app NEVER directly calls the scraper** — all requests go through the backend
2. Backend owns the mobile-facing API contract
3. Service-to-service communication between Backend and Scraper is internal only
4. JWT protects all user-specific endpoints

---

## Environment Configuration

### Backend (.env)
```env
PORT=5000
MONGO_URI=<mongodb-uri>
JWT_SECRET=<jwt-secret>
JWT_EXPIRES_IN=7d
GOOGLE_CLIENT_IDS=<comma-separated-client-ids>
GEMINI_API_KEY=<gemini-key>
SCRAPER_API_URL=http://localhost:5001
JOB_REFRESH_TTL_MINUTES=60
DEFAULT_JOB_RESULTS=20
JOB_RETENTION_DAYS=14
MAX_USER_JOBS=100
JOB_DESCRIPTION_MAX_LENGTH=1500
```

### Scraper
- Default URL: `http://localhost:5001`

---

## Development Commands

### Mobile App
```bash
cd Swipe-Select-MobileApp
npm install
npx expo start -c
```

### Backend
```bash
cd Swipe-Select-Backend
npm install
npm run dev
```

### Scraper
```bash
cd Web-Scrapper
pip install -r requirements.py
python app.py
```

---

## Onboarding Step Reference

| Step | Meaning |
|------|---------|
| 0 | New user, just registered |
| 1 | Resume extracted and profile populated |
| 2-12 | Onboarding in progress |
| 13 | Onboarding complete, can access dashboard |

---

## Important Design Decisions

1. **Profile/Preferences embedded in User:** Bounded data that reads together stays in User document
2. **Jobs/UserJobs separate:** Unbounded data lives in dedicated collections to avoid document bloat
3. **Remote filter not hard-coded:** Pakistan jobs don't reliably label remote, so filter is soft (for ranking/UI only)
4. **Job description capped:** 1500 character limit prevents storage bloat
5. **TTL indexes:** Jobs and user jobs auto-delete after 14 days

---

## Future Considerations

- Add Swagger docs for job API endpoints
- Add ranking logic using skill matching
- Add matches collection for apply acknowledgements
- Production service-to-service auth between backend and scraper if deployed separately

---

## Related Documentation

- [Backend AGENTS.md](./Swipe-Select-Backend/docs/AGENTS.md)
- [Mobile App Auth Setup](./Swipe-Select-MobileApp/docs/authentication%20.md)
- [Mobile App Onboarding](./Swipe-Select-MobileApp/docs/onboarding.md)
- [Scraper AGENTS.md](./Web-Scrapper/docs/AGENTS.md)