<div align="center">

# 🎯 HireSphere AI

**AI-Powered Recruitment & Employee Management System**

*Explainable resume screening with TF-IDF + Cosine Similarity — wrapped in a complete HR back office.*

Next.js 16 · TypeScript 5 · Prisma ORM · Tailwind CSS 4 · shadcn/ui · Three.js · Recharts

</div>

---

## 1. Project Overview

HireSphere AI is a full-stack web application that digitizes the complete hiring and employee-management lifecycle for a small/medium organization:

- **HR posts jobs** → **candidates register & upload PDF resumes** → the **AI engine scores every application** (TF-IDF text similarity + required-skill overlap, with evidence) → **HR reviews, shortlists, interviews, and hires** → **one click converts a hired candidate into an employee record** with portal access, leave balances and audit trail → **attendance, leave and performance are managed** in the same system.

Every AI score is **explainable and advisory**: HR sees exactly which skills matched, which didn't, the resume evidence for each match, the formula, and the algorithm version. **The system never auto-rejects a candidate** — every pipeline decision is a logged human action.

### Problem Statement
Traditional recruitment requires HR teams to manually review resumes, compare candidate skills with job requirements, shortlist applicants and maintain employee records across disconnected tools. This is slow, inconsistent and hard to audit. HireSphere AI centralizes the workflow and automates only the repetitive part — first-pass screening — while keeping all decisions with human reviewers.

### What this project demonstrates
Relational schema design · JWT auth + RBAC on every endpoint · file-upload security · a genuine (non-fake) NLP scoring pipeline · data-driven dashboards · REST API design · role-scoped reporting/export · responsive premium UI · 3D rendering with graceful fallbacks.

---

## 2. Features

### Public & Accounts
- Premium landing page with animated **3D hero** (Three.js, lazy-loaded, reduced-motion aware, ErrorBoundary fallback)
- Candidate registration, login/logout, forgot-password → reset-password flow (dev-mode token surfaced in UI; production path documented)
- bcrypt password hashing, JWT session in an **httpOnly cookie** (7-day expiry), suspended-account lockout
- Role-based access control enforced on **every API route** (UI hiding is never the security layer)

### Recruitment (HR / Admin)
- Job posting CRUD: draft → open → closed lifecycle, departments, salary range, openings, required/preferred skill chips, search/filter/sort/pagination
- Candidate pipeline with explicit status machine:
  `APPLIED → UNDER_REVIEW → SHORTLISTED → INTERVIEW → SELECTED → HIRED` (+ `REJECTED`, `WITHDRAWN`)
  Illegal transitions are rejected server-side; every change writes an immutable status-history row (actor, role, note, timestamp)
- **AI resume screening** (core feature — see §5)
- Job-specific **candidate ranking** with score sorting, min-score/skill/status filters and per-candidate evidence
- Interview scheduling (online/onsite/phone) that auto-advances the pipeline where legal
- Hired → Employee conversion: generates `HS-EMP-####` code, links the login, seeds leave balances, **idempotent on retry**
- Employee directory, attendance marking/correction (one row per employee/day enforced), leave approval with balance + overlap validation, performance reviews (1–5 scale), 7 exportable reports (JSON/CSV), print-friendly layout

### Candidate Portal
- Browse/search open jobs (title, skill, location, type, department), job detail pages
- PDF resume upload with drag & drop, live progress, 5 MB/PDF-type validation, real server-side text extraction, structured parsing (name/email/phone/education/experience/projects/certifications/skills) — **unparseable files fail honestly with the parser's message**
- Apply with resume → instant AI score snapshot; duplicate applications blocked (`409`), withdraw + re-apply supported
- Application tracking with full status timeline and AI snapshot (their own data only)

### Employee Portal
- Employment profile (code, department, designation, joining, probation state), leave balances
- Month attendance calendar with per-day status dots + records table
- Leave requests with inclusive-day counter, balance check, overlap detection, cancel-own-pending
- Performance review timeline (read-only for employees — by policy)

### Platform
- Data-driven dashboards for all roles (no hardcoded metrics anywhere)
- In-app notifications (application updates, interview scheduling, leave decisions, HR actions) with unread badge
- **Audit log** of every important action (actor, role, action, entity, details, IP, timestamp) — admin view with filters
- Dark (default) & light theme, glassmorphism accents, framer-motion micro-interactions, Recharts visualizations, fully responsive (mobile → desktop), sticky footer, loading/empty/error states everywhere

---

## 3. Screenshots

> Take these during your demo run — `docs/screenshots/` folder is referenced by the README.

| View | File |
|---|---|
| Landing page with 3D hero | `docs/screenshots/01-landing.png` |
| Login (demo credentials box) | `docs/screenshots/02-login.png` |
| HR dashboard (funnel + trends) | `docs/screenshots/03-hr-dashboard.png` |
| AI screening workspace | `docs/screenshots/04-ai-screening.png` |
| Candidate ranking | `docs/screenshots/05-ranking.png` |
| Candidate apply flow (instant score) | `docs/screenshots/06-apply.png` |
| Employee attendance calendar | `docs/screenshots/07-attendance.png` |

A screenshot helper script is included: `bun run src/scripts/capture-notes.ts` is not required — capture manually with any tool while logged in as each demo user.

## 4. Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| Frontend | Next.js 16 (App Router), React 19, TypeScript 5 | Single-page app at `/` with hash routing (`#/hr/jobs`, …) |
| UI | Tailwind CSS 4, shadcn/ui (New York), Lucide icons, framer-motion, sonner | Dark navy/indigo/violet + cyan accents |
| 3D | Three.js via @react-three/fiber | Lazy, `ssr:false`, ErrorBoundary + reduced-motion fallback |
| Charts | Recharts | Funnel, trends, department donut, attendance bars |
| Backend | Next.js Route Handlers (REST) | Same-language alternative to Flask blueprints; identical REST semantics |
| ORM/DB | Prisma ORM + SQLite | **MySQL-portable schema** (see `docs/database.md`) |
| Auth | bcryptjs + JWT (jose) in httpOnly cookie | Role & ownership checks server-side on every endpoint |
| AI | Custom TF-IDF + Cosine Similarity + skill dictionary (TypeScript) | Mirrors scikit-learn's `TfidfVectorizer` math (smooth IDF, L2 norm) |
| PDF | unpdf (pdf.js serverless build) for extraction; pdf-lib to generate synthetic demo resumes | Server-side only |
| Validation | Zod | One schema per endpoint payload |

> **Stack note (honest disclosure):** the original plan specified Flask + MySQL + scikit-learn. This build runs on a Node/Next.js runtime, so the same architecture is implemented with route handlers, Prisma and a TypeScript TF-IDF implementation. The algorithms are language-agnostic math; a Python port of `resume-matcher` maps 1:1 to `ai_engine/resume_matcher.py` in the plan. MySQL migration steps are documented in `docs/database.md`.

---

## 5. The AI Matching Engine (Core Feature)

**Location:** `src/lib/ai/` — `text-preprocessing.ts`, `skill-dictionary.ts`, `tfidf.ts`, `resume-matcher.ts`, `pdf-parser.ts` + `analyze-service.ts`

### Pipeline
```
PDF upload → validate (magic bytes, ≤5 MB, ≤20 pages) → unpdf text extraction
  → normalize → tokenize (stopwords, light stemming)
  → [A] TF-IDF vectorization (smooth IDF, L2 norm) → cosine similarity vs job text
  → [B] skill-dictionary scan → matched/missing required skills (+evidence snippets)
  → finalScore = w₁·tfidf + w₂·skillOverlap      (default w₁=0.6, w₂=0.4, env-configurable)
  → ResumeAnalysis row (scores, weights, evidence, algorithm version, timestamp)
```

### Scoring components
1. **`tfidfScore`** — cosine similarity between the resume text and the job text (title + description + skills). This is a **text similarity score**, *not* a probability of hiring success.
2. **`skillScore`** — required-skill overlap: `|required skills evidenced in resume| / |required skills|`. Each match carries the **exact resume sentence** that evidenced it. `null` when a job defines no required skills (weights then collapse to TF-IDF alone).
3. **`finalScore`** — weighted blend, stored with the weights used, so any historical score can be reproduced.

### Explainability guarantees
- HR sees: both component scores, matched skills **with resume evidence**, missing skills ("not evidenced"), extra skills, shared top terms, and a plain-language summary of the formula.
- `algorithmVersion` (`tfidf-cosine-v1.0`) is stored on every analysis.
- The UI displays a permanent banner: *AI scores are decision-support signals only — they never auto-reject.*
- Scoring uses **only** job-relevant text/skills. Age, gender, religion, caste, ethnicity, disability, photos and other protected characteristics are never modeled.

### Honesty rules implemented in code
- Image-only/encrypted/empty PDFs → explicit parse error states (no fabricated data).
- Fields that can't be reliably extracted are stored as `null` and displayed as "—".
- Scores are recomputable: `POST /api/ai/analyze/:applicationId` re-runs the pipeline.

---

## 6. Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Browser (SPA at /)                         │
│  Hash router → role-guarded views (Admin/HR/Candidate/Employee)│
│  Zustand stores · apiFetch wrapper · shadcn/ui · R3F hero      │
└───────────────▲────────────────────────────────────────────────┘
                │ fetch (same-origin, httpOnly cookie)
┌───────────────┴────────────────────────────────────────────────┐
│                Next.js Route Handlers (REST API)               │
│  /api/auth /api/jobs /api/applications /api/resumes /api/ai    │
│  /api/employees /api/attendance /api/leaves /api/performance   │
│  /api/dashboard /api/reports /api/notifications /api/audit-logs│
│  requireAuth(roles) → ownership checks → zod → Prisma → tx     │
│  rate limiting (login/register/upload) · audit · notify        │
└───────────────▲───────────────────────┬────────────────────────┘
                │                       │
      ┌─────────┴─────────┐   ┌─────────▼──────────────┐
      │  AI engine (lib)  │   │  Prisma ORM            │
      │  tfidf · matcher  │   │  17 models · FKs · idx │
      │  skill dictionary │   │  SQLite (MySQL-ready)  │
      │  pdf parser       │   └────────────────────────┘
      └───────────────────┘            ▲
                                       │
                        uploads/resumes/ (private, served
                        only via authorized download API)
```

Details: `docs/architecture.md` · Database: `docs/database.md` · API: `docs/api.md` + `docs/api-contract.md`

---

## 7. Installation & Getting Started (Windows PowerShell friendly)

```powershell
# 1. Clone and enter
git clone <your-repo-url> hiresphere-ai
cd hiresphere-ai

# 2. Install dependencies (Bun. npm/pnpm also work: npm install)
bun install

# 3. Configure environment
Copy-Item .env.example .env      # then edit if needed (see table below)

# 4. Create + push the database schema
bun run db:push

# 5. Seed demo data (synthetic resumes, real AI pipeline run)
bun run src/scripts/seed.ts

# 6. Start the dev server
bun run dev                       # → http://localhost:7000
```

### Environment variables (`.env`)

| Variable | Required | Default (dev) | Purpose |
|---|---|---|---|
| `DATABASE_URL` | ✅ | `file:./db/custom.db` | SQLite file path (or MySQL URL in production) |
| `SESSION_SECRET` | prod | dev fallback | HS256 signing key — **set a 32+ char random string in production** |
| `AI_WEIGHT_TFIDF` | – | `0.6` | TF-IDF weight in final score |
| `AI_WEIGHT_SKILL` | – | `0.4` | Skill-overlap weight |

### Demo accounts (created by the seed — password for all: `Password@123`)

| Role | Email |
|---|---|
| Admin | `admin@hiresphere.ai` |
| HR / Recruiter | `hr@hiresphere.ai` |
| Candidate | `priya.nair@example.com`, `rohan.mehta@example.com`, `sneha.kulkarni@example.com` |
| Employee (upgraded from hired candidate) | `aarav.sharma@example.com` |

> ⚠️ **These are development-only credentials.** Change them before any real use: log in as each account and reset via the app, or re-seed with your own values in `src/scripts/seed.ts`, or deactivate the accounts from Admin → User Management.

### Demo data
All demo people, jobs, applications, resumes, attendance, leaves and reviews are **synthetic** and produced by `src/scripts/seed.ts`. The seed generates five PDF resumes with pdf-lib and runs them through the *real* extraction + matching pipeline, so the scores you see are genuine.

---

## 8. Testing

The API was verified end-to-end with scripted curl suites (12 scenarios covering auth, RBAC negatives, ownership, duplicate-application prevention, upload validation, AI analysis, transitions, leave workflow, CSV export, password reset) and with full-browser E2E runs across all four roles. Details and the documented test plan live in **`docs/testing.md`**.

---

## 9. Deployment

Production checklist, Gunicorn-equivalent (Node standalone) setup, MySQL migration, HTTPS/CORS, health checks and troubleshooting: **`docs/deployment.md`**. The app exposes `GET /api/health` (checks DB connectivity) for platform health probes.

---

## 10. Limitations (honest list)

- **SQLite in this build, not MySQL.** The schema is portable and documented, but the running database is SQLite; production hardening (backups, concurrent writes) assumes MySQL.
- **No email service.** Password reset is fully implemented server-side; in this sandbox the reset token is surfaced in the UI (dev mode only). Wire SMTP/Resend for production email.
- **Resume parsing is heuristic.** Section detection + regex + a transparent skill dictionary. Unusual layouts can miss fields (rendered as "—", never guessed). Scanned/image-only PDFs cannot be parsed (explicit error).
- **TF-IDF is lexical.** It does not understand synonyms beyond the shipped dictionary or deep semantics ("builds APIs" vs "REST"). The roadmap's embeddings upgrade would address this.
- **In-memory rate limiting** — per-process, resets on restart. Use Redis for multi-instance production.
- **No automated CI test suite is committed.** Verification was done via documented curl/E2E scripts during development (see `docs/testing.md` for how to re-run them).
- **Notifications are in-app only** (no email/push).

## 11. Future Enhancements

Sentence-embedding semantic matching (local model) · AI-assisted interview-question generation from job requirements · email/push notifications · calendar integration for interviews · Docker Compose deployment ·_org SSO (OIDC)_ · advanced analytics (attrition, time-to-hire).

## 12. Individual Contribution (Academic)

This is an individual BCA final-year project. All layers — database design, REST API, AI engine, UI/UX, testing and documentation — were implemented by the author. Third-party libraries are limited to the framework/UI/math tooling listed in §4 and acknowledged in the source headers.

## 13. License

MIT — see `LICENSE`. You may use this code for learning and portfolios with attribution. Do not present it as a commercial product.

---

<div align="center">
<sub>Built as an academic full-stack + AI project · Explainable by design</sub>
</div>
#
