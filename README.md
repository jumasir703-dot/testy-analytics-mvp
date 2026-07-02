# Testy Analytics — School Exam Analysis System

A multi-tenant (multi-school) exam analytics platform supporting 8-4-4, CBC, IGCSE,
and custom grading systems. Built mobile-first / low-bandwidth-first for the
Kenyan/African school context. Billed per-school at **KES 5,000/year**, with
access locking automatically the moment a school's subscription lapses.

---

## 1. System Architecture

### 1.1 High-level modules

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│  Next.js PWA (works offline-ish, low-bandwidth, mobile-first)   │
│  - Admin/Principal dashboard   - Teacher mark-entry UI           │
│  - Parent/Student portal       - Public report-card viewer       │
└───────────────────────────┬───────────────────────────────────────┘
                            │ REST/JSON (+ optional GraphQL later)
┌───────────────────────────▼───────────────────────────────────────┐
│                        API LAYER (Node.js)                       │
│  Auth Module      | Schools & Users  | Students & Guardians       │
│  Academic Module  | Exams & Subjects | Marks/Results Module       │
│  Grading Engine   | Analytics Engine | Reports (PDF/Excel) Module │
│  Notifications    | Attendance       | Audit Logging              │
└───────────────────────────┬───────────────────────────────────────┘
                            │ Prisma ORM
┌───────────────────────────▼───────────────────────────────────────┐
│                     PostgreSQL (multi-tenant via schoolId)       │
└─────────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
        Object storage (S3-compatible)   Redis (cache/queues:
        for uploads, generated PDFs      SMS/email jobs, report
                                          generation jobs)
```

### 1.2 Multi-tenancy strategy

Single database, `schoolId` foreign key on every tenant-scoped table
(row-level isolation). Simpler to run cheaply on one Postgres instance,
which matters for cost-sensitive African-school deployments. Can graduate
to schema-per-tenant later if a school wants dedicated isolation.

### 1.3 Role model (RBAC)

| Role | Scope | Key permissions |
|---|---|---|
| `SUPER_ADMIN` | Platform | Manage schools (SaaS operator) |
| `SCHOOL_ADMIN` | One school | Full config: classes, subjects, grading, users |
| `HEAD_TEACHER` / `PRINCIPAL` | One school | Read-all analytics, approve report cards |
| `TEACHER` | Assigned subjects/classes | Enter marks for assigned subjects only |
| `CLASS_TEACHER` | One class/stream | Everything a teacher has + class-wide remarks |
| `PARENT` | Own children | View own children's results only |
| `STUDENT` | Self | View own results only (if school enables it) |

Enforced via JWT claims (`role`, `schoolId`, `staffId`) + middleware guards
+ row-level checks in queries (never trust the client for scoping).

### 1.4 Grading engine (curriculum-agnostic)

Instead of hardcoding "8-4-4" or "CBC", grading is modeled as a
**GradingSystem** with **GradeBands** (min score, max score, grade label,
points, remark). A school can have several grading systems active
(e.g., 8-4-4 for legacy classes, CBC rubric-based for lower grades).

- **8-4-4**: numeric 0–100 → A–E bands with points 1–12, mean grade computed from points.
- **CBC**: score → performance-level bands (BE, AE, ME, EE — Below/Approaching/Meeting/Exceeding
  Expectation), often per-strand/competency rather than a single subject score.
- **IGCSE**: numeric or raw-mark → A*–G bands, or 9–1 for some subjects.
- **Custom**: admin defines arbitrary bands.

This is modeled so the analytics engine only ever deals with **normalized
inputs**: `rawScore`, `maxScore`, `percentage`, `points`, `gradeLabel`.

### 1.5 Analytics engine

Computed both **on write** (denormalized aggregates updated when marks are
entered, for fast dashboard loads) and **on read** (ad-hoc queries for
custom filters). Key aggregates:
- Per student: subject trend over exams, overall mean, position in class/stream/grade.
- Per subject: mean, std dev, grade distribution, top/bottom performers.
- Per class/stream/school: mean score, mean grade, deviation vs school mean.
- Cross-cutting: gender analysis, subject-vs-subject correlation, improvement (Δ vs previous exam).

### 1.6 Reports

- **Report card (per student)**: subjects, marks, grade, position, teacher
  remarks, class teacher remark, principal remark, attendance summary.
- **Transcript**: multi-exam/multi-year history for one student.
- **Class/stream/school analysis report**: for head teacher/principal.

Rendered with a templating layer → HTML → PDF (Puppeteer/pdf-lib), queued
via Redis so bulk report generation (e.g., 500 report cards) doesn't block
the API.

### 1.7 Low-bandwidth / mobile-first considerations

- API responses paginated & filterable; heavy analytics precomputed.
- Bulk mark entry via CSV/Excel (small file, entered once offline in Excel,
  uploaded once — avoids per-cell network round trips).
- Frontend: Next.js with aggressive caching, small JS bundles, works
  reasonably on 3G; report cards can be viewed as lightweight HTML before
  PDF download.
- SMS (via Africa's Talking or similar) preferred over push notifications
  for parents without smartphones; email as secondary channel.

---

### 1.8 Subscription & auto-shutdown (Testy Analytics billing)

Each school has one `Subscription` row: `amount` (default KES 5,000), a
billing cycle (`startedAt` → `expiresAt`), and a `status`
(`TRIAL` / `ACTIVE` / `EXPIRED` / `CANCELLED`).

**How the shutdown works:** a `requireActiveSubscription` middleware sits in
front of every tenant route except `/api/auth` and `/api/billing`. On every
request it checks `expiresAt > now()` live against the database — there's no
reliance on a nightly cron to "notice" an expiry, so access locks the instant
the clock passes the paid-for date, not up to 24 hours later. The first
request after expiry also flips `status` to `EXPIRED` so dashboards reflect
it immediately.

A locked-out school gets `402 Payment Required` with the amount due and a
`renewUrl`, rather than a generic error — the frontend can catch that status
code anywhere in the app and show a "renew your subscription" screen instead
of a broken page.

`/api/billing` stays reachable while locked out, so a school admin can still:
- `GET /api/billing/status` — current plan, days remaining, payment history.
- `POST /api/billing/renew` — triggers an M-Pesa STK push (stubbed here;
  swap in a real Safaricom Daraja call in production).
- `POST /api/billing/confirm-payment` — records the payment and extends
  `expiresAt` by one year (extends from the *current* expiry if renewed
  early, so no paid time is lost; extends from *now* if already lapsed).

**Why per-school lockout instead of stopping the whole server:** this is a
multi-tenant system — one server can serve many schools. If School A's
payment lapses, School B (who is paid up) must keep working. If you deploy
one dedicated instance per school instead, you could additionally wire the
same expiry check into a process-level watchdog that calls `process.exit()`,
but that would take the database and any other tenants down with it, so it
isn't wired in by default here.



See `prisma/schema.prisma` for the full implementation. Summary of core
entities:

- **School** — tenant root.
- **User** — login identity (staff, parent, or student), linked to `Role`.
- **StaffProfile** / **ParentProfile** — role-specific extensions of `User`.
- **AcademicYear**, **Term** — time periods.
- **GradeLevel** (e.g., Form 1 / Grade 7) → **Stream** (e.g., 1 East) — class structure.
- **Student** — admission number, DOB, gender, guardian links, disability flag.
- **Subject**, **ClassSubject** (subject offered to a grade level, with teacher assignment).
- **GradingSystem**, **GradeBand** — curriculum-agnostic grading config.
- **Exam** — a sitting (e.g., "Term 2 Mid-Term 2026"), with weighting, applicable grade levels.
- **ExamSubject** — max score per subject per exam (can differ from default).
- **Result** — one student's raw score in one subject for one exam (the atomic mark).
- **Attendance** — daily/termly attendance record.
- **Remark** — teacher/class-teacher/principal comments on a result set.
- **Notification** — outbound SMS/email log.
- **AuditLog** — who changed what mark, when (critical for exam integrity disputes).
- **Subscription** / **Payment** — per-school billing: amount, expiry, status, and payment history.

---

## 3. API Structure (MVP)

```
POST   /api/auth/login
POST   /api/auth/refresh

GET    /api/billing/status                          (works even if subscription lapsed)
POST   /api/billing/renew                            (triggers M-Pesa STK push)
POST   /api/billing/confirm-payment                   (records payment, extends access)
PATCH  /api/billing/schools/:schoolId                 (SUPER_ADMIN: manual override)

GET    /api/schools/:id
PATCH  /api/schools/:id

GET    /api/students                 ?classId=&streamId=&search=&page=
POST   /api/students
POST   /api/students/bulk-import     (CSV/Excel)
GET    /api/students/:id
PATCH  /api/students/:id

GET    /api/subjects
POST   /api/subjects

GET    /api/exams
POST   /api/exams
GET    /api/exams/:id

GET    /api/results                  ?examId=&subjectId=&classId=
POST   /api/results                  (single mark entry, upsert)
POST   /api/results/bulk             (array of marks, or CSV/Excel upload)
GET    /api/results/student/:id

GET    /api/analytics/exam/:examId/subject/:subjectId
GET    /api/analytics/exam/:examId/class/:classId
GET    /api/analytics/student/:studentId/trend
GET    /api/analytics/school/:schoolId/overview

GET    /api/reports/report-card/:studentId/:examId   (PDF)
GET    /api/reports/class-analysis/:examId/:classId  (PDF)
```

All routes except `/auth/login` require `Authorization: Bearer <JWT>` and
are scoped to `req.user.schoolId`.

---

## 4. What's implemented in this MVP drop

1. ✅ Full Prisma schema (`prisma/schema.prisma`) — multi-curriculum ready.
2. ✅ Seed script with a demo school, grading systems (8-4-4, CBC, IGCSE), classes, students, subjects.
3. ✅ Express + TypeScript API server:
   - Auth (JWT, role middleware)
   - Students CRUD + bulk CSV/Excel import
   - Exams & subjects CRUD
   - Mark entry: single + bulk (with validation against max score, duplicate prevention)
   - Grading engine: auto-computes grade/points/remark from any active GradingSystem
   - Analytics: subject stats (mean, std dev, distribution), class ranking, student trend
4. 🔜 Next drops (as requested): React/Next.js dashboard, PDF report cards, CBC strand-level rubric UI, SMS notifications.

## 5. Running it

```bash
cd school-exam-system
cp .env.example .env          # set DATABASE_URL, JWT_SECRET
npm install
npx prisma migrate dev --name init
npm run seed
npm run dev                    # http://localhost:4000
```
