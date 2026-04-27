## Central agent instructions (Cursor / Gemini / Copilot)

This repo uses **one centralized set of AI-agent instructions**.

- **Source of truth**: this file (`.github/docs/AGENTS.md`)
- **Do not duplicate rules** in multiple places; other `AGENTS.md` files should only link here.

### Repo structure

- `Swipe-Select-Backend/`: Node/TypeScript backend
- `Swipe-Select-Frontend-Web/`: React + Vite + TypeScript web app
- `Swipe-Select-MobileApp/`: React Native / Expo (TypeScript)

### Ground rules for any agent

- Prefer **small, reviewable diffs** over large rewrites.
- Keep changes **scoped to the folder you’re working in** (backend vs web vs mobile).
- Never commit secrets (keys, tokens, `.env` files, credentials).
- Match existing tooling in each project (`eslint`, `prettier`, `tsconfig`) and keep formatting consistent.
- If you change behavior, also update or add the **minimum** tests or runnable checks available in that project.

### How to run (local)

Run commands from the relevant project directory:

- **Backend**
  - Install: `npm install`
  - Dev: `npm run dev` (or `npm start` if that’s what `package.json` uses)
  - Lint: `npm run lint` (if present)
  - Test: `npm test` (if present)

- **Frontend Web**
  - Install: `npm install`
  - Dev: `npm run dev`
  - Build: `npm run build`
  - Lint: `npm run lint` (if present)

- **Mobile**
  - Install: `npm install`
  - Start: `npm run start` (Expo)

If a script name differs, **use the scripts defined in that folder’s `package.json`**.

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

