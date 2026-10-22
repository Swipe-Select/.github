## Central agent instructions (Cursor / Gemini / Copilot)

This repo uses **one centralized set of AI-agent instructions**.

- **Source of truth**: this file (`.github/docs/AGENTS.md`)
- **Do not duplicate rules** in multiple places; other `AGENTS.md` files should only link here.

### Repo structure

- `Swipe-Select-Backend/`: Node/TypeScript backend (Express, MongoDB)
- `Swipe-Select-Frontend-Web/`: React + Vite + TypeScript web app
- `Swipe-Select-MobileApp/`: React Native / Expo (TypeScript)
- `Swipe-Select-ML/`: Python FastAPI ML service (ModernBERT, Qwen2.5 GGUF)
- `Web-Scrapper/`: Python Flask scraper service (JobSpy)

### Ground rules for any agent

- Prefer **small, reviewable diffs** over large rewrites.
- Keep changes **scoped to the folder you’re working in**.
- Never commit secrets (keys, tokens, `.env` files, credentials).
- Match existing tooling in each project (`eslint`, `prettier`, `tsconfig`, Python `requirements.txt`).
- If you change behavior, also update or add the **minimum** tests or runnable checks available in that project.

### How to run (local)

Run commands from the relevant project directory:

- **Backend (`Swipe-Select-Backend`)**
  - Install: `npm install`
  - Dev: `npm run dev`

- **Frontend Web (`Swipe-Select-Frontend-Web`)**
  - Install: `npm install`
  - Dev: `npm run dev`
  - Build: `npm run build`

- **Mobile (`Swipe-Select-MobileApp`)**
  - Install: `npm install`
  - Start: `npm start` (Expo)

- **ML Service (`Swipe-Select-ML`)**
  - Setup & Start: `./run.sh`
  - Verify: `python3 scripts/verify_setup.py`

- **Job Scraper (`Web-Scrapper`)**
  - Setup & Start: `./run_backend.sh`

If a script name differs, **use the scripts defined in that folder’s `package.json` or runner scripts**.

### Code expectations

- TypeScript: avoid `any` unless unavoidable; prefer narrow types.
- Errors: return helpful errors (backend) and avoid swallowing exceptions.
- Logging: don’t add noisy logs; keep logs structured if the project already does.
- API changes: if you change a request/response shape, update the client(s) that consume it.

### PR / review hygiene (if you’re working via PRs)

- Write PR descriptions with:
  - **Summary** (what + why)
  - **Test plan** (exact commands run)
  - **Screenshots** for UI changes (web/mobile)

