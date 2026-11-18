# CampusConnect — College Placement Portal

Full-stack placement portal: **Django REST Framework + SimpleJWT** backend, **React (Vite)** frontend.

It automates resume uploads, eligibility checks, job applications, company shortlisting and interview scheduling for three roles:

| Role | Can do |
|---|---|
| **Student** | Maintain profile, upload PDF resume, browse jobs with live eligibility feedback, apply, track status, see interview slots |
| **Company** | Post jobs with eligibility criteria, review applicants (profile + resume), shortlist / reject / select, schedule interviews |
| **Placement officer (TPO)** | Dashboard: placement rate, per-branch stats, all students, all applications, upcoming interviews |

## Quick start

Requires Python 3.10+ and Node 18+.

### Backend
```bash
cd backend
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python manage.py makemigrations accounts placement     # migrations are generated, not shipped
python manage.py migrate
python manage.py seed_demo                             # optional demo data
python manage.py runserver                             # http://127.0.0.1:8000
```

### Frontend
```bash
cd frontend
npm install
npm run dev                                            # http://localhost:5173 (proxies /api to Django)
```

### Demo logins (after `seed_demo`)
Password for all: `Passw0rd!demo`

| Username | Role | Notes |
|---|---|---|
| `tpo` | Placement officer (also Django admin superuser) | |
| `acme`, `globex` | Company | Acme: 2 jobs, Globex: 1 job |
| `asha` | Student | CSE, CGPA 8.9 — eligible for most things |
| `ravi`, `meera`, `nisha` | Student | Mixed branches / CGPAs |
| `karan` | Student | CGPA 6.2, 2 backlogs — good for seeing eligibility reasons |

Try it: log in as `asha` → apply to *Software Engineer* → log in as `acme` → *Applicants* → Shortlist → Schedule interview → log back in as `asha` to see the slot.

### Tests
```bash
cd backend && python manage.py test
```

## Architecture

```
backend/
  config/        settings, root urls
  accounts/      custom User (role), JWT login/refresh, registration, role permissions
  placement/     StudentProfile, Company, Job, Application, Interview
                 eligibility.py  – the eligibility rules (single source of truth)
                 views.py        – viewsets, resume endpoints, stats
                 tests.py        – API + rule tests
frontend/src/
  api.js         axios instance, JWT attach + automatic refresh-and-retry
  AuthContext    session state; ProtectedRoute logic lives in App.jsx
  pages/{student,company,tpo}/…
```

### Auth (JWT)
- `POST /api/auth/register/` (student or company; TPO accounts are created in Django admin / `createsuperuser`)
- `POST /api/auth/login/` → `{access, refresh, user}` · `POST /api/auth/refresh/` · `GET /api/auth/me/`
- Access token 30 min, refresh 24 h with rotation. The frontend refreshes transparently on a 401.

### Eligibility rules (`placement/eligibility.py`)
A student can apply only if **all** hold; the API returns *every* failed rule so the UI can explain it:
job is open · deadline not passed · not already placed · resume uploaded · profile has branch & graduation year · CGPA ≥ minimum · backlogs ≤ maximum · branch in allowed list (empty = all) · graduation year matches (if set).

The same function powers the job list badges, the `/jobs/{id}/eligibility/` endpoint, and the server-side check inside `apply` — so the UI can never bypass it.

### Application lifecycle
`applied → shortlisted → (interview) → selected | rejected`
- `interview` is set automatically when an interview is scheduled, and reverts to `shortlisted` if all interviews are cancelled.
- `selected` marks the student as placed, which blocks further applications.
- Selected / rejected are terminal.

### Interview scheduling
Only for shortlisted candidates of the company's own jobs. Must be in the future, 15–240 min. **Double-booking is prevented across companies**: a student can't be given two overlapping slots (back-to-back is fine). Rescheduling excludes the interview's own slot from the check.

### API summary
| Method & path | Who | Purpose |
|---|---|---|
| `GET/PATCH /api/students/me/` | student | profile |
| `GET/POST /api/students/me/resume/` | student | download / upload PDF (≤ 5 MB, magic-byte checked) |
| `GET /api/students/` | TPO | list students (`?branch=&placed=`) |
| `GET/POST/PATCH /api/jobs/` | all / company | role-scoped list; company creates & edits own |
| `POST /api/jobs/{id}/apply/` | student | apply (eligibility enforced) |
| `GET /api/jobs/{id}/eligibility/` | student | eligible + reasons |
| `GET /api/applications/?job=` | all (scoped) | applications |
| `POST /api/applications/{id}/status/` | company | `shortlisted` / `rejected` / `selected` |
| `GET /api/applications/{id}/resume/` | scoped | applicant resume (company sees only its own applicants) |
| `GET/POST/PATCH /api/interviews/` | all / company | schedule, reschedule, cancel |
| `GET /api/stats/` | TPO | dashboard numbers |

### Security notes
- Every queryset is scoped by role, so another company's jobs/applicants return 404 rather than leaking.
- Resumes are **not** served from a public `/media/` URL; they're streamed only through permission-checked endpoints, stored under random filenames.
- Students can't set `is_placed`; only a company's `selected` action does.

## Production notes / known limitations
- **CGPA and backlogs are self-reported.** A real deployment should let the TPO verify or enter them (add a `verified` flag and make those fields TPO-editable).
- Set `DJANGO_SECRET_KEY`, `DJANGO_DEBUG=0`, `DJANGO_ALLOWED_HOSTS`, `CORS_ALLOWED_ORIGINS`; switch SQLite → PostgreSQL; serve over HTTPS.
- JWTs are kept in `localStorage` for simplicity (readable by any XSS). For higher assurance, move the refresh token to an httpOnly cookie.
- Add rate limiting on login/register (DRF throttling) and pagination if lists grow large.
- No email/notification layer yet; students see interview slots in the portal only.
