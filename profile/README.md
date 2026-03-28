# LockInBro — ADHD-Aware Productivity System

## Technical Design Document — YHack 2026 Edition
**v2.0 · March 28–29, 2026 · Yale University, New Haven CT**
**Platforms:** macOS · iOS · iPadOS
**Stack:** Swift/SwiftUI · Python/FastAPI · PostgreSQL · Claude API · Hex API · L2CS-Net

---

## Target Tracks

| Track | Sponsor | Why LockInBro Fits | Prize |
|-------|---------|-------------------|-------|
| **Personal AI Agent** | Harper (YC-backed) | LockInBro is a fully autonomous AI agent: observes the user's screen, detects distraction, parses brain-dumps into tasks, generates context checkpoints, and proactively notifies. It automates the executive function that ADHD brains struggle with. | $2,000 + Meta Quest 3s + interview |
| **Societal Impact / Healthcare** | ASUS | ADHD affects 366M adults globally. LockInBro is a healthcare/accessibility tool that directly improves quality of life for a severely underserved population. | ASUS GX10 laptop + ZenScreen + ROG gear |
| **Best Use of Hex API** | Hex | LockInBro uses the Hex API to power its analytics dashboard: distraction pattern analysis, focus trend visualization, and weekly ADHD behavior reports. | $2,000 / $1,000 / $500 |

---

## 1. Executive Summary

LockInBro is an ADHD-aware personal AI agent that helps adults with ADHD manage tasks, resist distractions, and maintain focus through intelligent, real-time intervention. Built for the Apple ecosystem (macOS, iOS, iPadOS), it combines Claude's vision and language capabilities with on-device gaze tracking and Hex-powered analytics to deliver a system that truly adapts to the user's cognitive needs.

**Core thesis:** Instead of requiring the user to adapt to rigid productivity tools, LockInBro is a personal AI agent that adapts to the user — detecting distraction in real-time, parsing unstructured thoughts into actionable tasks, and gently nudging the user back to focus without shame or frustration.

**Why this matters:** 366 million adults worldwide have ADHD. Their brains' executive function systems work differently, making it hard to start tasks, stay on them, manage time, and recover from interruptions. Almost no software is built for cognitive accessibility — this is one of the largest underserved populations in tech.

---

## 2. Architecture Decisions

### Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **API Server** | Python / FastAPI on DigitalOcean droplet, port 3000 | Async-native, Pydantic validation, great for AI proxy work. Uvicorn uses ~40–60MB RAM — fits the 1GB droplet. |
| **Database** | PostgreSQL on same droplet, port 5432 | Already running. Database `focusapp` created with `root` superuser role. |
| **Auth** | Argon2 (email/password) + Sign in with Apple + JWT | Argon2id is the current best practice for password hashing (memory-hard, resistant to GPU/ASIC attacks). Apple Sign In for native iOS flow. JWT for stateless API auth. |
| **Reverse Proxy** | nginx with SSL via Certbot (Let's Encrypt) | Already configured on droplet, proxying `wahwa.com` to `localhost:3000`, WebSocket support enabled. |
| **Domain** | `wahwa.com` — API at `https://wahwa.com/api/v1/...` | Existing domain with SSL. Any service on port 3000 is automatically accessible at `www.wahwa.com`. |
| **AI Layer** | Claude API (Anthropic SDK) | Brain-dump parsing, screenshot VLM analysis, task step planning, context summarization. |
| **ML (On-Device)** | L2CS-Net via CoreML | Webcam-based gaze estimation for attention tracking on macOS. |
| **Analytics** | Hex API | Distraction pattern notebooks, focus trend dashboards, weekly ADHD behavior reports. Hex connects directly to Postgres. |
| **macOS App** | Swift / AppKit + SwiftUI | Menu bar agent, focus sessions, screenshot capture, eye tracking. |
| **iOS / iPadOS App** | SwiftUI + Screen Time API | Task management, brain-dump input, distraction app interception, notifications. |

### Why These Choices

- **Self-hosted over Railway/Vercel:** Vercel has timeout issues for VLM calls (10s default). Railway adds unnecessary deployment complexity when we already have infra. The droplet already has Postgres, nginx, SSL, and domain routing.
- **FastAPI over Hono:** Better ecosystem for AI/ML work (Anthropic SDK, numpy/pandas if needed for analytics). Pydantic gives instant request/response validation. Team has Python experience.
- **Argon2 over bcrypt:** Argon2id is memory-hard — more resistant to GPU cracking than bcrypt. The `argon2-cffi` Python package makes it trivial. Minimal CPU overhead on the 1GB droplet.
- **Separate steps table over JSONB:** The VLM auto-updates step statuses every ~20 seconds. With JSONB, every update requires read-modify-write of the entire array. With a separate table, it's a simple `UPDATE steps SET status = 'done' WHERE id = $1`. Cleaner for concurrent access.
- **No Supabase:** Adds external dependency when we already have everything locally.
- **No vector DB:** Postgres with JSONB is sufficient. LLM reasoning happens at call time via context window, not vector retrieval.
- **Hex over custom analytics:** No custom charting code needed. Iterative during hackathon — tweak queries in Hex UI without redeploying.

### Droplet Current State

- **OS:** Ubuntu (DigitalOcean, hostname `blindmaster-ubuntu`)
- **RAM:** 1GB (FastAPI ~40–60MB, Postgres already running)
- **Disk:** 25GB
- **Postgres:** Running on port 5432, database `focusapp` created
- **nginx:** Proxying `wahwa.com` and `www.wahwa.com` to `localhost:3000`, SSL via Certbot
- **PM2:** Killed (`pm2 kill`), `blinds_express` is stopped, port 3000 is free
- **Development:** SSH into server via VS Code Remote. Run Claude Code from local machine over SSH (don't install on droplet — saves RAM/disk).

---

## 3. System Architecture

### Data Flows

**Flow 1: Real-time Distraction Detection (macOS, <2s latency)**
- ScreenCaptureKit captures a screenshot every 15–30 seconds during an active focus session
- Screenshot sent as raw JPEG binary (quality 0.5, 1280x720) via multipart upload to `POST /distractions/analyze-screenshot` with full task + step context
- Claude Vision determines: which step user is on, if any steps completed, if user is distracted
- Backend auto-updates step statuses + writes `checkpoint_note` on the active step
- If distraction detected and confidence > 0.7, returns gentle nudge → local notification fires
- Distraction event logged to DB (app name, type, confidence — no screenshot stored)
- Screenshot discarded from memory after analysis (never persisted). Transient binary, GCs quickly on 1GB box.

**Flow 2: Brain-Dump Task Parsing (iOS, async)**
- User speaks or types a stream-of-consciousness dump
- If voice: on-device Speech framework transcribes to text
- Text sent to `POST /tasks/brain-dump` → Claude extracts structured tasks
- Returns parsed tasks with inferred deadlines, priorities, time estimates
- App asks "want me to break this into steps?" → `POST /tasks/{id}/plan` → Claude generates 5–15 min steps
- User reviews, edits if needed, confirms

**Flow 3: Focus Session + Auto-Update (macOS)**
- User starts session → `POST /sessions/start` → returns full context (task title, steps with statuses, current step, overall goal)
- Desktop captures periodic screenshots → VLM analyzes → auto-updates step statuses + checkpoint_note
- On interruption, checkpoint saved → `POST /sessions/{id}/checkpoint`
- On return → `GET /sessions/{id}/resume` → AI-generated context card: "You were on 'Write methods section' — you got through 3 of 5 paragraphs. Pick up at the results paragraph."

**Flow 4: iPad Focus Session (iPadOS, app-based detection)**
- User starts focus session on iPad → `POST /sessions/start` with `platform: "ipad"` + work app whitelist, OR joins existing Mac session via `POST /sessions/{id}/join` (triggered by tapping Live Activity / push notification / Handoff icon)
- Live Activity starts on lock screen showing current step, progress, and "Open [Work App]" button — persists for entire session, updated via ActivityKit push as Mac VLM detects step changes
- DeviceActivityMonitor configured: threshold = 2 min cumulative on non-whitelisted apps
- User works in whitelisted apps (Notes, Pages, etc.) — no network calls during on-task time
- If user drifts to non-whitelisted app for >2 min → monitor extension fires → local notification with gentle nudge
- Distraction logged via `POST /distractions/app-activity` (app name + duration, no screenshot)
- Step progress tracked via Live Activity "Mark Done" button + periodic check-in notifications + optional voice checkpoint notes
- Step updates written to same `steps` table via same endpoints — task list stays unified across Mac + iPad
- On return from distraction → `GET /sessions/{id}/resume` → same AI resume card flow as macOS

**Flow 5: Analytics Pipeline (Hex API, async/batch)**
- Backend writes distraction events, session data, and task completion data to Postgres
- Hex notebooks query Postgres directly via DB connection
- Hex computes: distraction frequency by app, focus duration trends, peak productivity windows, task completion rates
- Results fetched from Hex API and displayed in the iOS/iPad/macOS dashboard
- Weekly summary report generated as a Hex notebook run and surfaced as a push notification

### Privacy Architecture

| Data Type | Storage Policy | Encryption |
|-----------|---------------|------------|
| Screenshots (macOS only) | NEVER persisted. Sent as raw JPEG binary via multipart upload, analyzed in-memory, discarded after VLM response. | TLS in transit only |
| App activity (iPad) | Only app name + duration sent to backend during focus sessions. No screenshots, no screen content. | TLS in transit |
| Gaze data | On-device only. Raw coordinates never leave the Mac. Only aggregate attention scores sync. | Keychain-encrypted local store |
| Task data | Synced to backend, stored in Postgres. | TLS in transit (nginx SSL) |
| Distraction events | Aggregated: app name, type, confidence, VLM summary text. No images. | TLS in transit |
| Voice recordings | On-device transcription via Apple Speech. Audio discarded post-transcription. | Never stored |
| Analytics (Hex) | Aggregated, anonymized event data. No PII in Hex notebooks. | Hex platform encryption |

**Future (post-hackathon):** Replace Claude API with on-device model (Apple foundation models or quantized VLM) to eliminate screenshot transmission entirely.

---

## 4. Database Schema

```sql
-- Users (dual auth: email/password with Argon2 + Apple Sign In)
CREATE TABLE public.users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT UNIQUE,
    password_hash   TEXT,                    -- Argon2id hash (null for Apple-only users)
    apple_user_id   TEXT UNIQUE,             -- Apple Sign In subject ID (null for email-only users)
    display_name    TEXT,
    timezone        TEXT DEFAULT 'America/Chicago',
    distraction_apps TEXT[] DEFAULT '{}',
    device_tokens   JSONB DEFAULT '[]',      -- APNs push tokens: [{"platform": "ipad", "token": "..."}]
    preferences     JSONB DEFAULT '{}',      -- notification schedule, energy windows, etc.
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now(),
    CONSTRAINT auth_method CHECK (email IS NOT NULL OR apple_user_id IS NOT NULL)
);

-- Tasks
CREATE TABLE public.tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    description     TEXT,
    priority        INT DEFAULT 0,          -- 0=unset, 1=low, 2=med, 3=high, 4=urgent
    status          TEXT DEFAULT 'pending',  -- pending | planning | ready | in_progress | done | deferred
    deadline        TIMESTAMPTZ,
    estimated_minutes INT,
    source          TEXT DEFAULT 'manual',   -- manual | brain_dump | voice
    tags            TEXT[] DEFAULT '{}',
    plan_type       TEXT,                    -- llm_generated | user_defined | hybrid | null
    brain_dump_raw  TEXT,                    -- original dump text for context
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Steps (separate table — VLM updates individual steps every ~20s)
CREATE TABLE public.steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES public.tasks(id) ON DELETE CASCADE,
    sort_order      INT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    estimated_minutes INT,
    status          TEXT DEFAULT 'pending',  -- pending | in_progress | done | skipped
    checkpoint_note TEXT,                    -- VLM-written progress context within the step
                                            -- e.g., "wrote 3 of 5 paragraphs, results paragraph next"
    last_checked_at TIMESTAMPTZ,            -- last VLM observation timestamp
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Focus Sessions
CREATE TABLE public.sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    task_id         UUID REFERENCES public.tasks(id) ON DELETE SET NULL,
    platform        TEXT NOT NULL,            -- mac | ipad | iphone
    started_at      TIMESTAMPTZ DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    status          TEXT DEFAULT 'active',   -- active | completed | interrupted
    checkpoint      JSONB DEFAULT '{}',
    -- checkpoint shape: {current_step_id, last_action_summary,
    --                     next_up, goal, active_app, last_screenshot_analysis,
    --                     attention_score, distraction_count,
    --                     work_app_bundle_ids (iPad only)}
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Distraction Events (Hex queries this table directly)
CREATE TABLE public.distractions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES public.sessions(id) ON DELETE SET NULL,
    detected_at     TIMESTAMPTZ DEFAULT now(),
    distraction_type TEXT,          -- app_switch | off_screen | idle | browsing
    app_name        TEXT,
    duration_seconds INT,
    confidence      FLOAT,
    vlm_summary     TEXT,
    nudge_shown     BOOLEAN DEFAULT false
);

-- Distraction Patterns (for Hex analytics)
CREATE TABLE public.distraction_patterns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    pattern_type    TEXT,           -- time_of_day | app_sequence | task_avoidance
    description     TEXT,
    frequency       INT DEFAULT 1,
    last_seen       TIMESTAMPTZ DEFAULT now(),
    metadata        JSONB DEFAULT '{}'
);

-- Indexes
CREATE INDEX idx_tasks_user ON tasks(user_id, status);
CREATE INDEX idx_steps_task ON steps(task_id, sort_order);
CREATE INDEX idx_steps_status ON steps(task_id, status);
CREATE INDEX idx_sessions_user ON sessions(user_id, started_at DESC);
CREATE INDEX idx_sessions_active ON sessions(user_id, status) WHERE status = 'active';
CREATE INDEX idx_distractions_user ON distractions(user_id, detected_at DESC);
CREATE INDEX idx_distractions_app ON distractions(user_id, app_name, detected_at DESC);
CREATE INDEX idx_distractions_hourly ON distractions(user_id, EXTRACT(HOUR FROM detected_at));
```

### Step Architecture: Why No JSONB, Why No Splitting

**Separate table over JSONB:** The VLM auto-updates step statuses every ~20 seconds during a focus session. With JSONB, every update requires reading the full array, finding the step, modifying it, and writing the entire blob back. With a separate table, it's `UPDATE steps SET status = 'done', completed_at = now() WHERE id = $1`. Cleaner, faster, no race conditions.

**No dynamic step splitting:** When a user is halfway through a step, we do NOT split it into sub-steps. For ADHD users, the task list growing while they're working feels punishing and creates decision paralysis. The whole point of 5–15 minute steps is that they're small enough to be atomic.

**Instead: `checkpoint_note` for within-step progress.** The VLM writes natural language into `checkpoint_note` every time it analyzes a screenshot — things like "wrote 3 of 5 paragraphs" or "has the data table open but hasn't started filling it in." The step stays as one item with status `in_progress`, but carries rich context about exactly where the user left off.

**Resume flow uses `checkpoint_note`:** When the user comes back, the context-resume prompt gets the `checkpoint_note` and generates: "You were on 'Write methods section' — you got through 3 of 5 paragraphs. Pick up at the results paragraph." Hyper-specific context without structural complexity.

---

## 5. API Endpoints

Base URL: `https://wahwa.com/api/v1`
All endpoints (except auth) require `Authorization: Bearer <jwt>` header.
All timestamps are ISO 8601 UTC.

### Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Email + password → Argon2id hash → create user → return JWT |
| POST | `/auth/login` | Email + password → verify Argon2 hash → return JWT |
| POST | `/auth/apple` | Exchange Apple identity token → find or create user → return JWT |
| POST | `/auth/refresh` | Refresh expired JWT using refresh token |

### Tasks

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tasks` | List tasks, filterable by status/priority, sortable by priority/deadline |
| GET | `/tasks/upcoming` | Tasks with deadlines in next 48 hours |
| POST | `/tasks` | Create a task manually |
| POST | `/tasks/brain-dump` | Raw text/voice input → Claude parses into structured tasks with priorities, deadlines, estimates |
| POST | `/tasks/{id}/plan` | Generate step-by-step plan (Claude breaks into 5–15 min ADHD-friendly steps) |
| PATCH | `/tasks/{id}` | Update task fields (partial) |
| DELETE | `/tasks/{id}` | Soft-delete a task |

### Steps

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tasks/{id}/steps` | List all steps for a task (ordered by sort_order) |
| PATCH | `/steps/{id}` | Update step fields (status, checkpoint_note, etc.) |
| POST | `/steps/{id}/complete` | Mark step as done with completed_at timestamp |

### Sessions

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/sessions/start` | Start focus session, optionally linked to a task. Returns full task context + all steps. |
| POST | `/sessions/{id}/checkpoint` | Save context checkpoint (current step, last action, attention score, active app) |
| POST | `/sessions/{id}/end` | End session with status (completed / interrupted) |
| POST | `/sessions/{id}/join` | Second device joins an active session. Returns full task context + suggested work app. Avoids duplicate sessions. |
| GET | `/sessions/{id}/resume` | Get last checkpoint + step checkpoint_notes → AI-generated resume card |

### Distraction Detection

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/distractions/analyze-screenshot` | Desktop sends screenshot (raw JPEG binary via multipart) + task/step context. Claude Vision analyzes. Auto-updates step statuses + checkpoint_note. Returns nudge if distracted. |
| POST | `/distractions/app-check` | Lightweight mobile pre-launch intercept. Checks distraction list, returns pending task count + most urgent task. |
| POST | `/distractions/app-activity` | iPad/iPhone reports app-based distraction event (app name, duration). Logs distraction, returns nudge using task context. No screenshot needed. |

### Analytics (Hex-Powered)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/analytics/distractions` | Top distracting apps, hourly patterns, recovery times (Hex notebook) |
| GET | `/analytics/focus-trends` | Daily focus minutes, completion rate, attention trends (Hex notebook) |
| GET | `/analytics/weekly-report` | AI-generated weekly ADHD behavior summary (Hex notebook) |
| POST | `/analytics/refresh` | Force-refresh all analytics (triggers fresh Hex notebook runs) |
| GET | `/analytics/summary` | Quick stats for dashboard (direct Postgres query, no Hex) |

---

## 6. Key Request/Response Schemas

### POST /auth/register

**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword",
  "display_name": "Alex",
  "timezone": "America/New_York"
}
```

**Response (201):**
```json
{
  "access_token": "jwt_string",
  "refresh_token": "opaque_string",
  "expires_in": 3600,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "display_name": "Alex",
    "created_at": "ISO 8601"
  }
}
```

### POST /auth/apple

**Request:**
```json
{
  "identity_token": "Apple ID JWT string",
  "authorization_code": "string",
  "full_name": "string | null (first sign-in only)"
}
```

**Response (200):** Same shape as register response.

### POST /tasks/brain-dump

**Request:**
```json
{
  "raw_text": "I need to email Sarah about the project, oh and I have that dentist appointment Thursday, also gotta buy groceries, the presentation is due Friday...",
  "source": "voice",
  "timezone": "America/Chicago"
}
```

**Response (200):**
```json
{
  "parsed_tasks": [
    {
      "title": "Email Sarah about the project",
      "description": "Discuss project timeline",
      "priority": 3,
      "deadline": "2026-04-03T17:00:00Z",
      "estimated_minutes": 15,
      "source": "brain_dump",
      "tags": ["work"]
    },
    {
      "title": "Dentist appointment",
      "priority": 3,
      "deadline": "2026-04-02T10:00:00Z",
      "estimated_minutes": null,
      "source": "brain_dump",
      "tags": ["health"]
    }
  ],
  "unparseable_fragments": [
    "also that thing from last week"
  ],
  "ask_for_plans": true
}
```

### POST /tasks/{id}/plan

**Request:**
```json
{
  "plan_type": "llm_generated"
}
```

**Response (200):**
```json
{
  "task_id": "uuid",
  "plan_type": "llm_generated",
  "steps": [
    {
      "id": "uuid",
      "sort_order": 1,
      "title": "Open email client and find Sarah's thread",
      "estimated_minutes": 2,
      "status": "pending",
      "checkpoint_note": null
    },
    {
      "id": "uuid",
      "sort_order": 2,
      "title": "Draft update on project timeline",
      "estimated_minutes": 8,
      "status": "pending",
      "checkpoint_note": null
    },
    {
      "id": "uuid",
      "sort_order": 3,
      "title": "Review and send",
      "estimated_minutes": 3,
      "status": "pending",
      "checkpoint_note": null
    }
  ]
}
```

### POST /distractions/analyze-screenshot

**Request:** multipart/form-data
```
screenshot: raw JPEG binary (UploadFile, 1280x720, quality 0.5)
window_title: string
session_id: UUID
task_context: JSON string containing:
  {
    "task_title": "Finish presentation",
    "task_goal": "Complete Q1 report slides for Monday meeting",
    "steps": [
      {"id": "uuid", "sort_order": 1, "title": "Open template", "status": "done"},
      {"id": "uuid", "sort_order": 2, "title": "Write methods section", "status": "in_progress",
       "checkpoint_note": "wrote intro and background paragraphs"},
      {"id": "uuid", "sort_order": 3, "title": "Create data tables", "status": "pending"}
    ]
  }
```

**Response (200):**
```json
{
  "on_task": false,
  "current_step_id": "uuid-of-step-2",
  "checkpoint_note_update": "wrote intro and background paragraphs, results paragraph is next",
  "steps_completed": ["uuid-of-step-1"],
  "distraction_type": "browsing",
  "app_name": "Safari",
  "confidence": 0.92,
  "gentle_nudge": "Looks like Twitter pulled you in! You were making great progress on the methods section — just the results paragraph left.",
  "vlm_summary": "User has Safari open with Twitter feed visible. VS Code in background with presentation file."
}
```

**Backend side-effects on this response:**
1. `UPDATE steps SET status = 'done', completed_at = now() WHERE id IN (completed step UUIDs)`
2. `UPDATE steps SET checkpoint_note = '...', last_checked_at = now() WHERE id = current_step_id`
3. `INSERT INTO distractions (...)` if `on_task = false`

### POST /distractions/app-check

**Request:**
```json
{
  "app_bundle_id": "com.instagram.ios"
}
```

**Response (200):**
```json
{
  "is_distraction_app": true,
  "pending_task_count": 3,
  "most_urgent_task": {
    "title": "Finish presentation",
    "priority": 4,
    "deadline": "2026-04-03T17:00:00Z",
    "current_step": "Write methods section",
    "steps_remaining": 4
  },
  "nudge": "Hey, quick check-in! You have 3 tasks waiting. Top priority: Finish presentation (due tomorrow)."
}
```

### GET /sessions/{id}/resume

**Response (200):**
```json
{
  "session_id": "uuid",
  "task": {
    "title": "Finish presentation",
    "overall_goal": "Complete Q1 report slides for Monday meeting"
  },
  "current_step": {
    "id": "uuid",
    "title": "Write methods section",
    "checkpoint_note": "wrote intro and background paragraphs, results paragraph is next",
    "last_checked_at": "ISO 8601"
  },
  "progress": {
    "completed": 2,
    "total": 5,
    "attention_score": 72,
    "distraction_count": 2
  },
  "resume_card": {
    "welcome_back": "Welcome back! You were making great progress.",
    "you_were_doing": "You were on 'Write methods section' — got through the intro and background paragraphs.",
    "next_step": "Pick up at the results paragraph. Should take about 5 more minutes.",
    "motivation": "You're 2 of 5 steps done — the hardest part is behind you!"
  }
}
```

### POST /sessions/start

**Request:**
```json
{
  "task_id": "uuid (optional — session can be taskless)",
  "platform": "mac | ipad | iphone (default: mac)",
  "work_app_bundle_ids": ["com.apple.notes", "com.apple.Pages"]
}
```
`work_app_bundle_ids` is optional, used for iPad sessions to configure DeviceActivityMonitor whitelist.

**Response (201):**
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "task_id": "uuid | null",
  "platform": "mac",
  "started_at": "ISO 8601",
  "ended_at": null,
  "status": "active",
  "checkpoint": {},
  "created_at": "ISO 8601"
}
```

**Backend side-effects:**
1. Enforces one active session per user — returns 409 if one already exists
2. Sends push notification to user's other devices (APNs)
3. Sends ActivityKit push to start Live Activity on iPad (if session started on Mac)

### POST /sessions/{id}/checkpoint

**Request:**
```json
{
  "current_step_id": "uuid (optional)",
  "last_action_summary": "string (optional)",
  "next_up": "string (optional)",
  "goal": "string (optional)",
  "active_app": "string (optional)",
  "last_screenshot_analysis": "string (optional)",
  "attention_score": 72,
  "distraction_count": 2
}
```
All fields optional — checkpoint is a JSONB merge, so only provided fields are updated.

**Response (200):** Same `SessionOut` shape as start.

### POST /sessions/{id}/end

**Request:**
```json
{
  "status": "completed | interrupted (default: completed)"
}
```

**Response (200):** Same `SessionOut` shape with `ended_at` set and status updated.

**Backend side-effects:**
1. Sets `ended_at = now()`
2. If other devices are joined, sends push: "Session ended on {platform}. Keep focusing?"
3. Dismisses Live Activity on iPad via ActivityKit push

### PATCH /tasks/{id}

**Request (all fields optional):**
```json
{
  "title": "Updated title",
  "description": "Updated description",
  "priority": 3,
  "status": "in_progress",
  "deadline": "ISO 8601",
  "estimated_minutes": 45,
  "tags": ["work", "urgent"]
}
```

**Response (200):** Full `TaskOut` with updated fields.

### PATCH /steps/{id}

**Request (all fields optional):**
```json
{
  "status": "in_progress | done | skipped",
  "checkpoint_note": "wrote 3 of 5 paragraphs, results section is next"
}
```

**Response (200):**
```json
{
  "id": "uuid",
  "task_id": "uuid",
  "sort_order": 2,
  "title": "Write methods section",
  "description": null,
  "estimated_minutes": 10,
  "status": "in_progress",
  "checkpoint_note": "wrote 3 of 5 paragraphs, results section is next",
  "last_checked_at": "ISO 8601",
  "completed_at": null,
  "created_at": "ISO 8601"
}
```

Used by iPad manual step updates and voice checkpoint notes. Same endpoint the VLM auto-update flow writes to server-side.

### POST /steps/{id}/complete

**Request:** None (empty body).

**Response (200):** Same `StepOut` shape with `status: "done"` and `completed_at` set.

Convenience endpoint — equivalent to `PATCH /steps/{id}` with `{"status": "done"}` but also sets `completed_at = now()`.

### Device Token Registration

**Not yet wired to an endpoint.** The backend has `push.register_device_token(user_id, platform, token)` in `services/push.py` but no router calls it. **Needs a new endpoint:**

```
POST /auth/device-token
{
  "platform": "mac | ipad | iphone",
  "token": "APNs device token hex string"
}
```

Each device calls this on app launch / token refresh. Stored in `users.device_tokens` JSONB as `[{"platform": "ipad", "token": "..."}]`. One token per platform — upserts on conflict.

### Deep Link URL Scheme

FocusFlow registers the URL scheme `focusflow://` for cross-device deep links:

| URL | Triggered By | Action |
|-----|-------------|--------|
| `focusflow://join-session?id={session_id}&open={url_encoded_app_scheme}` | Live Activity tap / Push notification tap | Join session + chain-open target app |
| `focusflow://resume-session?id={session_id}` | Push notification (session resume) | Open resume card for session |
| `focusflow://task?id={task_id}` | Push notification (deadline, morning brief) | Open task detail view |

The `open` parameter is URL-encoded (e.g., `mobilenotes%3A%2F%2F`). The `onOpenURL` handler decodes it, calls `UIApplication.shared.open()` to chain-open the target app, and joins the session in background.

---

## 7. AI Prompt Engineering

All AI calls route through the backend's `services/llm.py` module for prompt versioning, rate limiting, and cost tracking.

### Prompt Design Principles

- **Non-judgmental tone:** Never use words like "you got distracted again" or "you wasted time." Frame everything as collaborative support.
- **Concise outputs:** ADHD users have limited working memory. Keep AI responses short, scannable, and actionable.
- **Structured JSON:** All AI responses are JSON for reliable parsing. No free-text responses in production flows.
- **Context-aware:** Every prompt includes current task, step progress, time of day, and recent history.
- **Graceful degradation:** If Claude API is down, fall back to simple heuristic-based responses.
- **ADHD-friendly steps:** Generated steps should be 5–15 minutes each. First step should be the easiest (reduce activation energy). Context restoration messages should be warm and specific.

### Brain-Dump Parsing Prompt

```
System: You are a task parser for someone with ADHD.
Extract structured tasks from this brain dump.
Be generous with deadlines - infer from context.
If no deadline is obvious, set priority to 0 (unset).
Today's date: {current_date}
User's timezone: {timezone}

Respond ONLY with JSON, no other text:
{
  "parsed_tasks": [{
    "title": "concise task title",
    "description": "any extra detail from the dump",
    "deadline": "ISO 8601 or null",
    "priority": "0-4 integer (0=unset, 1=low, 2=med, 3=high, 4=urgent)",
    "estimated_minutes": "number or null",
    "tags": ["work", "personal", "health", "errands", etc.]
  }],
  "unparseable_fragments": ["text that couldn't be parsed into tasks"]
}
```

### Step Planning Prompt

```
System: You are an ADHD-friendly task planner.
Break this task into concrete steps of 5-15 minutes each.
Each step should be specific enough that someone with ADHD
can start immediately without decision paralysis.

Rules:
- First step should be the EASIEST (reduce activation energy)
- Steps should be independently completable
- Include time estimates per step
- Total estimated time should roughly match the task estimate
- No step longer than 15 minutes

Task: {task_title}
Description: {task_description}
Estimated total: {estimated_minutes} minutes

Respond ONLY with JSON array:
[{
  "sort_order": 1,
  "title": "specific action description",
  "description": "additional detail if needed",
  "estimated_minutes": number
}]
```

### Distraction Detection Prompt (Claude Vision)

```
System: You are a focus assistant analyzing a user's screen.
The user's current task and step progress:
  Task: {task_title}
  Goal: {task_goal}
  Steps:
{formatted_steps_with_statuses_and_checkpoint_notes}
  Window title reported by OS: {window_title}

Analyze this screenshot. Determine:
1. Is the user working on their task or distracted?
2. Which step are they currently on?
3. Have any steps been completed since last check?
4. What specific progress have they made within the current step?

Respond ONLY with JSON:
{
  "on_task": boolean,
  "current_step_id": "step UUID if identifiable, null otherwise",
  "checkpoint_note_update": "1-2 sentence description of within-step progress (e.g., 'wrote 3 of 5 paragraphs, results section is next'). null if can't determine.",
  "steps_completed": ["UUIDs of steps that appear done based on screenshot"],
  "distraction_type": "app_switch | browsing | idle | null",
  "app_name": "primary visible application",
  "confidence": 0.0-1.0,
  "gentle_nudge": "kind, non-judgmental 1-sentence reminder if distracted. Include specific context about their progress to make returning easier. null if on-task.",
  "vlm_summary": "1-sentence factual description of what's visible on screen"
}
```

### Context Resume Prompt

```
System: Generate a brief, encouraging context-resume card for
someone with ADHD returning to their task.
Be warm, specific, and action-oriented. No shame. No generic platitudes.
Use the checkpoint_note to give hyper-specific context about where they left off.

Inputs:
  - Task: {task_title}
  - Overall goal: {goal}
  - Current step: {current_step_title}
  - Current step checkpoint_note: {checkpoint_note}
  - Steps completed: {completed_count} of {total_count}
  - Next step after current: {next_step_title}
  - Time away: {minutes_away} minutes
  - Attention score before leaving: {attention_score}

Respond ONLY with JSON:
{
  "welcome_back": "short friendly greeting (max 8 words)",
  "you_were_doing": "1 sentence referencing checkpoint_note specifically",
  "next_step": "concrete next action with time estimate",
  "motivation": "1 sentence encouragement (ADHD-friendly, no shame)"
}
```

### Task Prioritization Prompt

```
System: You are an ADHD-friendly task prioritizer.
Consider: deadlines, estimated effort, task dependencies,
and the user's energy patterns.

Rules:
- Hard deadlines always take top priority
- Front-load quick wins (<15min) for momentum
- Group errands together
- Deprioritize tasks with no deadline and low urgency

Input: {tasks_json}
Current time: {now}
User's timezone: {timezone}

Respond ONLY with JSON array:
[{
  "task_id": "uuid",
  "recommended_priority": 1-4,
  "reason": "1-sentence explanation"
}]
```

---

## 8. Hex API Integration

Hex is a collaborative data platform for analytics notebooks. LockInBro uses the Hex API to power its analytics layer — distraction pattern analysis, focus trend visualization, and personalized ADHD behavior insights.

### Why Hex?

- **No custom charting code:** Hex notebooks handle data transformation, aggregation, and visualization. We just fetch results via API.
- **Iterative analysis:** During the hackathon, tweak analytics queries in the Hex UI without redeploying the backend.
- **Shareable reports:** Weekly ADHD behavior reports generated as Hex notebook runs.
- **SQL + Python:** Hex notebooks can run SQL against our Postgres data and Python for ML-based pattern detection.

### Architecture: Hex in the Stack

Hex connects directly to the `focusapp` Postgres database on the DigitalOcean droplet. Ensure port 5432 is accessible to Hex (add to firewall rules or something). The backend triggers notebook runs via the Hex API, and clients fetch computed analytics.

```
Client Apps          Backend (FastAPI)           PostgreSQL (droplet)
  │                       │                        │
  │  POST /distractions   │   INSERT events         │
  │ ────›                 │ ────›                   │
  │  POST /sessions/end   │   INSERT session         │
  │ ────›                 │ ────›                   │
  │                       │                        │
  │                       Hex API                    │
  │                       │                        │
  │                       │  SQL queries against  ──›│
  │                       │  focusapp DB via Hex     │
  │                       │  data connection         │
  │                       │                        │
  │  GET /analytics/*     │  Trigger Hex notebook    │
  │ ────›                 │  run, return computed    │
  │ ‹── JSON results      │  results                │
```

### Hex Notebooks

**Notebook 1: Distraction Pattern Analysis**

```sql
-- Top distracting apps (last 7 days)
SELECT
    app_name,
    COUNT(*) as distraction_count,
    ROUND(AVG(confidence)::numeric, 2) as avg_confidence,
    distraction_type
FROM distractions
WHERE user_id = {{user_id}}
  AND detected_at > NOW() - INTERVAL '7 days'
GROUP BY app_name, distraction_type
ORDER BY distraction_count DESC
LIMIT 10;
```

```sql
-- Distraction frequency by hour of day
SELECT
    EXTRACT(HOUR FROM detected_at) as hour_of_day,
    COUNT(*) as distraction_count,
    ROUND(AVG(confidence)::numeric, 2) as avg_conf
FROM distractions
WHERE user_id = {{user_id}}
  AND detected_at > NOW() - INTERVAL '30 days'
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

```python
# Recovery time analysis
import pandas as pd

df = dataframe_1  # distractions
sessions = dataframe_2  # focus sessions

df['recovery_minutes'] = (
    df['next_session_start'] - df['detected_at']
).dt.total_seconds() / 60

summary = {
    'avg_recovery_min': round(df['recovery_minutes'].mean(), 1),
    'median_recovery_min': round(df['recovery_minutes'].median(), 1),
    'worst_distractor': df.groupby('app_name')['recovery_minutes'].mean().idxmax()
}
```

**Notebook 2: Focus Trend Dashboard**

```sql
-- Daily focus summary (last 30 days)
SELECT
    DATE(started_at) as focus_date,
    COUNT(*) as sessions,
    SUM(EXTRACT(EPOCH FROM (ended_at - started_at))) / 60 as total_focus_minutes,
    ROUND(
        COUNT(*) FILTER (WHERE status = 'completed')::numeric
        / NULLIF(COUNT(*), 0) * 100, 1
    ) as completion_pct
FROM sessions
WHERE user_id = {{user_id}}
  AND started_at > NOW() - INTERVAL '30 days'
GROUP BY focus_date
ORDER BY focus_date;
```

**Notebook 3: Weekly ADHD Behavior Report (Stretch)**

```python
# Generate weekly summary via Claude
import anthropic

weekly_data = {
    'focus_minutes': focus_df['total_minutes'].sum(),
    'sessions_completed': sessions_df[sessions_df.status == 'completed'].shape[0],
    'top_distractors': distraction_df.head(3).to_dict(),
    'best_focus_hours': hourly_df.nlargest(3, 'session_count'),
    'tasks_completed': tasks_df[tasks_df.status == 'done'].shape[0],
    'tasks_created': len(tasks_df),
}

client = anthropic.Anthropic()
report = client.messages.create(
    model='claude-sonnet-4-20250514',
    max_tokens=500,
    messages=[{
        'role': 'user',
        'content': f'''Generate an ADHD-friendly weekly report.
        Be encouraging, highlight wins, gently note patterns.
        Keep it under 200 words. Data: {weekly_data}'''
    }]
)
```

### Hex API Integration Code

```python
# backend/app/services/hex_service.py
import httpx
from app.config import settings

HEX_API_BASE = "https://app.hex.tech/api/v1"
HEADERS = {
    "Authorization": f"Bearer {settings.HEX_API_TOKEN}",
    "Content-Type": "application/json",
}

NOTEBOOKS = {
    "distraction_patterns": settings.HEX_NB_DISTRACTIONS,
    "focus_trends": settings.HEX_NB_FOCUS_TRENDS,
    "weekly_report": settings.HEX_NB_WEEKLY_REPORT,
}


async def run_notebook(notebook_key: str, user_id: str) -> dict:
    """Trigger a Hex notebook run with user params."""
    project_id = NOTEBOOKS[notebook_key]
    async with httpx.AsyncClient() as client:
        # Trigger run
        resp = await client.post(
            f"{HEX_API_BASE}/project/{project_id}/run",
            headers=HEADERS,
            json={"inputParams": {"user_id": user_id}},
        )
        run_id = resp.json()["runId"]

        # Poll for completion
        import asyncio
        while True:
            status = await client.get(
                f"{HEX_API_BASE}/project/{project_id}/run/{run_id}",
                headers=HEADERS,
            )
            data = status.json()
            if data["status"] == "COMPLETED":
                return data["resultData"]
            if data["status"] == "ERRORED":
                raise Exception(f"Hex run failed: {data['error']}")
            await asyncio.sleep(2)
```

### Hex Setup Steps

1. Create Hex account (free tier)
2. Add Postgres data connection: point to droplet IP, port 5432, database `focusapp`. (May need to open port 5432 in UFW.)
3. Create 3 notebooks using the SQL/Python above
4. Copy project IDs to backend `.env`:
   ```
   HEX_NB_DISTRACTIONS=nb_xxxxx
   HEX_NB_FOCUS_TRENDS=nb_yyyyy
   HEX_NB_WEEKLY_REPORT=nb_zzzzz
   HEX_API_TOKEN=hex_xxxxx
   ```
5. Generate API token: Hex Settings → API

---

## 9. macOS Desktop Application

### App Architecture

| Component | Framework | Description |
|-----------|-----------|-------------|
| Menu Bar Agent | AppKit (NSStatusItem) | Persistent menu bar icon showing focus state. Quick-access to start/stop sessions, view stats. |
| Focus Session Window | SwiftUI | Floating panel showing current task, current step, step progress, timer, attention score, distraction count. |
| Screenshot Engine | ScreenCaptureKit | Periodic screen capture during active sessions. Configurable interval (15–60s). Resolution: 1280x720, JPEG quality 0.5. Sent as raw binary (not base64) to minimize bandwidth (~40–60 KB per frame). |
| Gaze Tracker | AVFoundation + CoreML | L2CS-Net model converted to CoreML. Tracks where user is looking on screen. |
| Process Monitor | NSWorkspace | Observes frontmost app changes. Auto-detects work apps and suggests starting a focus session. |
| Notification Engine | UserNotifications | Delivers distraction alerts with gentle nudges, context reminders, and session prompts. |

### Distraction Detection Pipeline

1. **Trigger:** Timer fires every N seconds (default 20s) during active focus session
2. **Capture:** ScreenCaptureKit captures primary display at 1280x720
3. **Pre-filter:** Compare with previous screenshot using perceptual hash. If >95% similar, skip API call to save cost.
4. **Analyze:** Send screenshot as raw JPEG binary via multipart upload to `POST /distractions/analyze-screenshot` with full task + step context (including checkpoint_notes)
5. **Auto-update:** Backend receives VLM response. Updates step statuses + writes new `checkpoint_note` on the active step. Updates session checkpoint.
6. **Act:** If distracted and confidence > 0.7, fire notification with gentle nudge (nudge references specific checkpoint_note progress). If 0.5–0.7, log but don't interrupt.
7. **Discard:** Delete screenshot from memory. Never write to disk.

### Eye Tracking Module (L2CS-Net)

- **Model:** L2CS-Net (ResNet backbone) converted to CoreML via coremltools. Input: 224x224 face crop. Output: yaw + pitch angles.
- **Face Detection:** Apple Vision framework (VNDetectFaceRectanglesRequest) to locate and crop face before passing to L2CS-Net.
- **Gaze Mapping:** Convert yaw/pitch to approximate screen coordinates via 4-corner calibration step (user looks at 4 corners of screen during setup).
- **Attention Score:** Rolling 60-second window. If gaze leaves screen area for >40% of window, score drops. Score: 0–100.
- **Privacy:** Camera feed processed in-memory only. No frames saved. Live indicator when camera is active.

**CoreML Conversion (one-time build step — do first):**
```python
import coremltools as ct
import torch
from l2cs import Pipeline

model = Pipeline(weights='L2CSNet_gaze360.pkl')
traced = torch.jit.trace(model.net, torch.randn(1, 3, 224, 224))
mlmodel = ct.convert(traced,
    inputs=[ct.TensorType(shape=(1, 3, 224, 224))],
    minimum_deployment_target=ct.target.macOS14
)
mlmodel.save('L2CSNet.mlpackage')
```

### Focus Sessions & Context Checkpointing

**Session Lifecycle:**
1. **Detection:** Process monitor detects a work app (VS Code, Xcode, Figma) as frontmost for >60 seconds.
2. **Prompt:** Gentle notification: "Looks like you're starting work. Want to start a focus session?"
3. **Setup:** User selects a task (or app suggests one based on priority). Task steps displayed with current progress.
4. **Active:** Screenshot monitoring + eye tracking active. VLM auto-updates step statuses and checkpoint_notes every 20s.
5. **Interruption:** If user switches away, checkpoint saved via `POST /sessions/{id}/checkpoint`.
6. **Resume:** When user returns → `GET /sessions/{id}/resume` → AI-generated context card using checkpoint_note: "You were on 'Write methods section' — got through the intro and background paragraphs. Pick up at the results paragraph."
7. **Complete:** Session ended via `POST /sessions/{id}/end`. Summary shown. Data synced.

---

## 10. iOS / iPadOS Application

### Core Screens

| Screen | Purpose | Key Components |
|--------|---------|---------------|
| Brain Dump | Capture unstructured thoughts via voice or text | Large text field, mic button, "Parse Tasks" action, preview of extracted tasks, "Break into steps?" prompt |
| Task Board | View and manage tasks with smart prioritization | Priority-sorted list (0–4 scale), swipe actions, due date badges, step progress bars, focus-time estimates |
| Dashboard | Distraction patterns and focus analytics (Hex-powered) | Weekly charts, top distractions, focus streak, attention trend, Hex-generated insights |
| Settings | Configure distraction apps, notification preferences | App blocklist for pre-launch intercept, notification schedule |

### Brain-Dump Task Parser (UX Flow)

1. User taps the brain-dump button (prominent, always accessible)
2. Speaks freely: "I need to email Sarah about the project, oh and I have that dentist appointment Thursday, also gotta buy groceries, the presentation is due Friday..."
3. On-device Speech framework transcribes in real-time (user sees live text)
4. User taps "Parse" → text goes to `POST /tasks/brain-dump`
5. Returns structured tasks with inferred metadata
6. App asks: "Want me to break these into steps?" → `POST /tasks/{id}/plan` for each
7. User reviews steps, edits if needed, confirms

### Distraction App Interception

**iOS approach (Screen Time API):**
- FamilyControls: Request authorization to manage user's own device (personal use, not parental control)
- ManagedSettings: Apply ShieldConfiguration to selected apps the user chose to guard
- Custom shield shows current top-priority task + step progress + encouraging message
- User can dismiss with 5-second hold (friction, not punishment)

```swift
struct LockInBroShield: ShieldConfigurationDataSource {
    override func configuration(
        shielding app: Application
    ) -> ShieldConfiguration {
        let topTask = TaskStore.shared.topPriority()
        let stepProgress = TaskStore.shared.stepProgress(for: topTask)
        return ShieldConfiguration(
            backgroundBlurStyle: .systemUltraThinMaterial,
            title: .init(text: "Hey, quick check-in!", color: .accent),
            subtitle: .init(
                text: "Your top task: \(topTask.title)\n\(stepProgress)"
            ),
            primaryButtonLabel: .init(
                text: "Back to Focus",
                color: .white, backgroundColor: .accent
            ),
            secondaryButtonLabel: .init(
                text: "Hold 5s to open anyway",
                color: .secondary
            )
        )
    }
}
```

### iPad Focus Sessions

iPad is a productive device — users take notes, write papers, complete homework. Unlike iPhone (passive task management) and macOS (screenshot VLM analysis), iPad gets its own focus session model based on **active app monitoring + timed check-ins**.

**Why not screenshots?** iPadOS sandboxing prevents apps from capturing other apps' screens. No ScreenCaptureKit equivalent exists. Instead, we use the Screen Time framework (`DeviceActivityMonitor`) to detect when the user drifts to non-work apps, and periodic check-in prompts for step progress.

#### Detection Model

| Signal | API | How It Works |
|--------|-----|-------------|
| User left work apps | DeviceActivityMonitor | Configure a `DeviceActivitySchedule` for the focus session duration. Set monitored apps = everything EXCEPT the user's whitelisted work apps. When cumulative non-whitelisted usage exceeds threshold (default 2 min), the monitor extension fires. |
| User left FocusFlow | UIApplication lifecycle | `sceneDidEnterBackground` starts a local timer. If user doesn't return within threshold, schedule nudge notification. |
| Distraction app launched | FamilyControls + ShieldConfiguration | Same shield system as iPhone — already in spec. Works during iPad focus sessions too. |

#### Work App Whitelist

When starting a focus session on iPad, the user picks which apps are "on-task" for this session:

- **AI-suggested:** Backend infers likely work apps from the task title/description (e.g., "Write essay" → Notes, Pages, Google Docs, Safari)
- **User-curated:** User can add/remove from the suggestion
- **Remembered:** Whitelist saved per-task, reused on future sessions

```
POST /sessions/start
{
  "task_id": "uuid",
  "platform": "ipad",
  "work_app_bundle_ids": ["com.apple.notes", "com.apple.Pages", "com.google.docs"]
}
```

#### Distraction Detection Pipeline (iPad)

1. **Start:** User starts focus session → app configures `DeviceActivityMonitor` with threshold (default 2 min off-task)
2. **Monitor:** DeviceActivityMonitor tracks cumulative time on non-whitelisted apps
3. **Threshold hit:** Monitor extension fires → sends local notification with gentle nudge + current step context
4. **Log:** App sends distraction event to `POST /distractions/app-activity` with app name + duration
5. **Return:** User taps notification → app shows resume card via `GET /sessions/{id}/resume`

#### Step Progress Tracking (iPad)

Without VLM, step progress uses a **periodic check-in + manual update** model:

| Mechanism | How | Frequency |
|-----------|-----|-----------|
| Live Activity | Lock screen widget showing current step title, progress bar, + "Mark Done" button. Same Live Activity used for cross-device handoff — updated via ActivityKit push when Mac VLM detects step changes. | Always visible during session |
| Periodic check-in | Local notification: "How's '{step_title}' going? Tap to mark done or update." | Every 10–15 min (configurable) |
| Voice update | User can voice-dictate a quick checkpoint note (on-device Speech → text saved as `checkpoint_note`) | On demand |
| Time-based suggestion | If user spends ≥ `estimated_minutes` in whitelisted apps, suggest: "Looks like you've been at '{step_title}' for a while — ready to mark it done?" | Triggered by timer |

The backend writes `checkpoint_note` updates and step status changes through the same endpoints the macOS VLM flow uses — the task list stays unified across devices.

#### New Endpoint: POST /distractions/app-activity

Lightweight endpoint for iPad (and iPhone) to report app-based distraction events without screenshots:

```
POST /distractions/app-activity
{
  "session_id": "uuid",
  "app_bundle_id": "com.instagram.ios",
  "app_name": "Instagram",
  "duration_seconds": 145,
  "returned_to_task": true
}
```

**Response (200):**
```json
{
  "distraction_logged": true,
  "session_distraction_count": 3,
  "gentle_nudge": "Instagram grabbed your attention for a couple minutes. No worries — you were on 'Write intro paragraph'. Pick up where you left off!"
}
```

**Backend side-effects:**
1. `INSERT INTO distractions (...)` with `distraction_type = 'app_switch'`
2. Update session checkpoint `distraction_count`
3. Generate nudge using task + step + checkpoint_note context (Claude text, not Vision — cheap)

### Cross-Device Session Handoff

When a user starts a focus session on one device, other devices can join that session — so the task list, step progress, and distraction tracking stay unified. The primary use case: user researches on Mac, takes notes on iPad.

#### How It Works: Push Notification + Live Activity + Deep Link Chain

The core challenge on iPadOS: a background app can't silently open another app. No Shortcuts workaround reliably solves this (Shortcuts can't dynamically choose a target app, and "Open App" doesn't execute while the device is locked). Instead, we use a **push notification as the initial alert** and a **Live Activity as the persistent, always-visible prompt** — both requiring just one tap to reach the target app.

**Why Live Activity is the right mechanism:**

| | Push Notification | Live Activity |
|---|---|---|
| Visible on lock screen | Until dismissed/cleared | **Always, for entire focus session** |
| User picks up iPad 20 min later | Notification might be buried | **Still right there on lock screen** |
| Dynamic content updates | Need new notification each time | **Updates in place** (step changes, progress) |
| Tap behavior | Opens FocusFlow → chains to target app | Same — but persistent visibility is the key difference |

A focus session IS an ongoing activity — exactly what Live Activities were designed for.

#### Handoff Flow

1. **Mac starts session** → `POST /sessions/start` with `platform: "mac"`
2. **Backend sends two things to iPad via APNs:**
   - **Push notification** (initial alert): "Focus session active: 'Take notes on Q1 report'. Tap to join on iPad." **[Join on iPad]** — **[Dismiss]**
   - **ActivityKit push** (starts Live Activity): persistent lock screen widget showing task title + current step + "Open Notes" — stays visible for the entire session
3. **Mac advertises `NSUserActivity`** with task context — iPad shows Handoff icon on lock screen / dock as a tertiary entry point
4. **User picks up iPad** → sees Live Activity on lock screen (or the push notification, or Handoff icon) → **taps once**
5. **Deep link chain fires** — FocusFlow opens via deep link → `onOpenURL` handler immediately calls `UIApplication.shared.open(targetURL)` **before UI renders** → target app (Notes, Pages, etc.) opens. User sees: tap → Notes opens. FocusFlow flashes for a fraction of a second or not at all.
6. **FocusFlow joins session in background** — `POST /sessions/{id}/join` called during the deep link handler. DeviceActivityMonitor configured with work app whitelist. Live Activity continues showing on lock screen.
7. **Both devices now feed into the same session** — Mac VLM updates steps, iPad tracks app usage + manual step completions. All writes go to the same `steps` table.

#### The Deep Link Chain (Key Implementation Detail)

The trick that makes this feel seamless: handle the deep link and chain-open the target app **before your own UI renders.** The user taps the Live Activity → Notes opens. FocusFlow is just a passthrough.

```swift
// FocusFlow deep link handler — fires before UI renders
.onOpenURL { url in
    // url = focusflow://join-session?id=xxx&open=mobilenotes%3A%2F%2F
    if let targetScheme = url.queryParam("open"),
       let targetURL = URL(string: targetScheme) {
        // Chain-open immediately — target app opens before FocusFlow UI appears
        UIApplication.shared.open(targetURL)
        // Join session in background (non-blocking)
        Task { await SessionManager.shared.joinSession(from: url) }
    }
}
```

#### Live Activity: Lock Screen Widget

The Live Activity stays on the iPad lock screen for the entire focus session, updated in real-time via ActivityKit push notifications as the Mac's VLM detects step progress:

```
┌─────────────────────────────────────────────┐
│  📝 Focus: Take notes on Q1 report          │
│  Current step: Write methods section         │
│  Progress: ██████░░░░ 3/5 steps              │
│                                              │
│              [ Open Notes ]                  │
└─────────────────────────────────────────────┘
```

- **"Open Notes" button** → deep link with target app scheme → chain-open
- **Step/progress updates** → backend sends ActivityKit push when VLM updates step status on Mac
- **Session ends** → Live Activity dismissed automatically

```swift
// ActivityKit: define the Live Activity attributes
struct FocusSessionAttributes: ActivityAttributes {
    let sessionId: String
    let taskTitle: String

    struct ContentState: Codable, Hashable {
        let currentStepTitle: String
        let completedSteps: Int
        let totalSteps: Int
        let suggestedAppScheme: String
        let suggestedAppName: String
    }
}
```

#### NSUserActivity Setup (macOS Side)

Handoff provides a tertiary entry point — the Handoff icon on iPad's lock screen / dock. Less prominent than the Live Activity, but works even if ActivityKit push fails.

```swift
let activity = NSUserActivity(activityType: "com.focusflow.focus-session")
activity.title = "Focus: \(task.title)"
activity.userInfo = [
    "session_id": session.id.uuidString,
    "task_id": task.id.uuidString,
    "current_step_id": currentStep.id.uuidString,
    "suggested_app_scheme": "mobilenotes://"
]
activity.isEligibleForHandoff = true
activity.becomeCurrent()
```

#### New Endpoint: POST /sessions/{id}/join

Allows a second device to join an existing active session instead of creating a duplicate:

```json
// Request
{
  "platform": "ipad",
  "work_app_bundle_ids": ["com.apple.notes", "com.apple.Pages"]
}
```

```json
// Response (200)
{
  "session_id": "uuid",
  "joined": true,
  "task": {
    "id": "uuid",
    "title": "Take notes on Q1 report webpage",
    "goal": "Summarize key findings from the Q1 data"
  },
  "current_step": {
    "id": "uuid",
    "title": "Read through methodology section",
    "status": "in_progress",
    "checkpoint_note": "skimmed intro, starting on data tables"
  },
  "all_steps": [ /* full step list with statuses */ ],
  "suggested_app_scheme": "mobilenotes://",
  "suggested_app_name": "Notes"
}
```

**Backend side-effects:**
1. Add `platform` entry to session checkpoint: `checkpoint.devices = ["mac", "ipad"]`
2. Store iPad's `work_app_bundle_ids` in checkpoint for DeviceActivityMonitor config
3. Push notifications and ActivityKit updates for this session now route to both devices

#### Multi-Device Session Rules

| Scenario | Behavior |
|----------|----------|
| Mac ends session while iPad is joined | iPad Live Activity updates: "Session ended on Mac. Keep focusing?" with [Keep Going] / [End Session]. If keep going, session stays active with iPad as primary. |
| iPad joins, then user goes off-task on iPad | Normal iPad distraction flow (DeviceActivityMonitor → nudge). Mac session unaffected. |
| Both devices update the same step simultaneously | Last-write-wins on `checkpoint_note` (fine — Mac VLM writes every 20s, iPad writes on manual action). Step `status` transitions are idempotent (pending→in_progress→done). |
| User starts a NEW session on iPad for same task | Backend detects active session for this task, prompts: "You have an active session on Mac. Join that one instead?" |
| Mac VLM completes a step | Backend sends ActivityKit push → iPad Live Activity updates in place (progress bar advances, current step changes). |

### Smart Notifications

| Type | Trigger | Example |
|------|---------|---------|
| Deadline Approaching | Task deadline within 24h / 1h | "Heads up — 'Presentation' is due tomorrow. ~2hrs estimated. 3 steps left. Want to start a session?" |
| Morning Brief | Daily at user-configured time | "Good morning! You have 3 tasks today. Top priority: Email Sarah. First step: Open email client (~2 min)." |
| Focus Streak | 3+ consecutive sessions completed | "3 focus sessions today! You're on a roll." |
| Gentle Nudge | No task activity for 4+ hours | "Just checking in. Your task list has 2 urgent items. No pressure!" |
| Cross-Device Handoff | Focus session started on another device | Push: "Focus session active on Mac: 'Take notes on Q1 report'. Tap to join." + Live Activity starts on lock screen with task/step/progress and "Open Notes" button. |
| Session Ended on Other Device | Joined session ended by primary device | Live Activity updates: "Session ended on Mac. Keep focusing?" [Keep Going] / [End Session] |

---

## 11. Project Structure

```
focusflow/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app entry, port 3000
│   │   ├── config.py            # Settings (DATABASE_URL, JWT_SECRET, ANTHROPIC_API_KEY, HEX_*)
│   │   ├── middleware/
│   │   │   └── auth.py          # JWT validation + Argon2 utils
│   │   ├── routers/
│   │   │   ├── auth.py          # register + login + apple
│   │   │   ├── tasks.py         # CRUD + brain-dump + plan
│   │   │   ├── steps.py         # step updates
│   │   │   ├── sessions.py      # start + checkpoint + end + resume + join
│   │   │   ├── distractions.py  # analyze-screenshot + app-check + app-activity
│   │   │   └── analytics.py     # Hex-powered endpoints
│   │   ├── services/
│   │   │   ├── llm.py           # Claude API wrapper (all prompts)
│   │   │   ├── hex_service.py   # Hex API client
│   │   │   ├── db.py            # asyncpg Postgres client
│   │   │   └── push.py          # APNs client (push notifications + ActivityKit push for Live Activity updates)
│   │   ├── models.py            # Pydantic schemas
│   │   └── types.py
│   ├── requirements.txt         # fastapi, uvicorn, asyncpg, argon2-cffi,
│   │                            # anthropic, httpx, python-jose, python-multipart
│   ├── alembic/                 # DB migrations
│   └── .env                     # DATABASE_URL, JWT_SECRET, ANTHROPIC_API_KEY, HEX_*
├── ios/
│   └── LockInBro/
│       ├── LockInBroApp.swift
│       ├── Views/
│       │   ├── BrainDumpView.swift
│       │   ├── TaskBoardView.swift
│       │   ├── DashboardView.swift     # Hex analytics display
│       │   ├── SettingsView.swift
│       │   └── iPad/
│       │       ├── iPadFocusSessionView.swift  # Focus session UI for iPad
│       │       └── WorkAppPickerView.swift      # Whitelist picker at session start
│       ├── LiveActivity/
│       │   ├── FocusSessionLiveActivity.swift   # Lock screen widget UI (step progress, "Open Notes" / "Mark Done")
│       │   └── FocusSessionAttributes.swift     # ActivityKit attributes + content state
│       ├── Models/
│       ├── Services/
│       │   ├── APIClient.swift
│       │   ├── SpeechService.swift
│       │   ├── ScreenTimeManager.swift
│       │   ├── DeviceActivityManager.swift  # iPad: configures DeviceActivityMonitor for focus sessions
│       │   └── HandoffReceiver.swift        # Handles NSUserActivity from Mac + session join
│       └── Shared/
├── macos/
│   └── LockInBroMac/
│       ├── LockInBroMacApp.swift
│       ├── MenuBar/
│       │   └── StatusBarController.swift
│       ├── FocusSession/
│       │   ├── SessionWindow.swift
│       │   ├── ScreenshotEngine.swift
│       │   ├── DistractionDetector.swift
│       │   ├── ContextCheckpoint.swift
│       │   └── HandoffAdvertiser.swift      # Publishes NSUserActivity for cross-device handoff
│       ├── EyeTracking/
│       │   ├── GazeTracker.swift
│       │   └── L2CSNet.mlpackage
│       └── ProcessMonitor/
│           └── AppWatcher.swift
└── shared/
    ├── Models/                   # Shared Swift models (Task, Step, Session)
    └── Networking/               # Shared API client
```

---

## 12. Hackathon Scope & Demo Strategy

### Team Split

| Person | Owns | Key Focus |
|--------|------|-----------|
| **Person 1 (Backend)** | FastAPI, Postgres, Claude API proxy, Hex notebooks, deployment on droplet | Get `/tasks/brain-dump` + `/tasks/{id}/plan` working in first 3 hours. Hex running by hour 8. |
| **Person 2 (Desktop)** | macOS app: menu bar, focus session, ScreenCaptureKit, process monitor | Core distraction detection pipeline. Wire to `/analyze-screenshot` by hour 6. |
| **Person 3 (Eye Tracking + iOS)** | L2CS-Net → CoreML, gaze tracker, iOS app (brain dump, task board, dashboard) | CoreML conversion in first 30 min. iOS brain dump view by hour 4. |

### MVP (Must Ship)

| Feature | Owner | Effort | Track Impact |
|---------|-------|--------|-------------|
| Brain-dump → tasks + step planning | P1 + P3 | Medium | Harper (AI Agent) + ASUS (Healthcare) |
| Distraction detection (screenshot + VLM + auto-step-update + checkpoint_note) | P1 + P2 | Medium | Harper (AI Agent) — core differentiator |
| Focus session with context checkpoint + resume using checkpoint_note | P1 + P2 | Medium | Harper (AI Agent) — shows context awareness |
| Task board with step progress | P3 | Low | Foundation for everything |
| Hex analytics dashboard | P1 + P3 | Medium | Hex API track — must demo prominently |
| Auth (Argon2 + Apple Sign In + JWT) | P1 | Low | Required infrastructure |

### Stretch Goals

| Feature | Owner | Effort | Track Impact |
|---------|-------|--------|-------------|
| Eye tracking attention score | P2 + P3 | High | Harper + ASUS — impressive tech demo |
| Distraction app shield (Screen Time API) | P3 | Medium | ASUS (Healthcare) — visual + impactful |
| Weekly AI behavior report (Hex) | P1 | Low | Hex track — deeper integration |
| Smart notifications | P3 | Low | ASUS (Healthcare) — proactive care |

### Build Priority Order

1. **Hour 0–1:** Postgres schema + migrations, auth endpoints, FastAPI scaffold, CoreML conversion
2. **Hour 1–3:** `POST /tasks/brain-dump` + `POST /tasks/{id}/plan` + `GET /tasks` + `GET /tasks/{id}/steps` — this is the demo centerpiece
3. **Hour 3–6:** `POST /sessions/start` + checkpoint + `POST /distractions/analyze-screenshot` (with step auto-update + checkpoint_note writes) + macOS screenshot engine
4. **Hour 6–8:** Wire screenshots to backend, distraction notifications, `GET /sessions/{id}/resume`, Hex setup + distraction_patterns notebook
5. **Hour 8–12:** iOS brain dump view + task board (with step progress bars), focus_trends Hex notebook, analytics endpoints
6. **Hour 12–18:** Integration testing, context resume flow end-to-end, dashboard UI, eye tracking integration
7. **Hour 18–22:** Stretch goals, bug fixes, UI polish
8. **Hour 22–23:** Record demo video, submit to Devpost

### Demo Script (3-minute pitch)

1. **[0:00–0:30] Problem + Stats:** "366M adults with ADHD. No software built for cognitive accessibility. We built LockInBro." *(ASUS track setup)*
2. **[0:30–1:15] Brain Dump → Steps (live):** Speak a messy thought stream into iPhone. Show Claude parsing into tasks, then breaking a task into 5–15 min steps. *(Harper track: AI agent)*
3. **[1:15–1:45] Focus Session (live):** Start a session on Mac with a task. Open Twitter mid-session. Show VLM catching it, auto-updating step progress via checkpoint_note, and firing a gentle nudge that references their exact progress. *(Harper track: autonomous agent)*
4. **[1:45–2:15] Context Resume:** Switch away, come back. Show the AI-generated context card: "You were on 'Write methods section' — got through the intro and background paragraphs. Pick up at the results paragraph." *(Harper track)*
5. **[2:15–2:45] Analytics Dashboard:** Show the Hex-powered dashboard: distraction patterns by hour, top distracting apps, focus trend chart. *(Hex track)*
6. **[2:45–3:00] Close:** "This is just the start. On-device VLM, Apple Watch, calendar integration. LockInBro adapts to you."

---

## 13. Cost Estimates

| Operation | Frequency | Cost/Call | Daily Cost |
|-----------|-----------|----------|------------|
| Screenshot analysis (Claude Vision) | ~120/day (1 per 20s, 40min focus) | ~$0.003 | $0.36 |
| Brain-dump parsing | ~3/day | ~$0.002 | $0.006 |
| Step planning | ~3/day | ~$0.002 | $0.006 |
| Context resume generation | ~5/day | ~$0.001 | $0.005 |
| Task prioritization | ~2/day | ~$0.002 | $0.004 |
| Hex API (notebook runs) | ~10/day | Free tier | $0.00 |
| **Total per user** | | | **~$0.38/day** |

Perceptual hash pre-filter can reduce Vision calls by ~40%, bringing daily cost to ~$0.23/user. Hex free tier covers hackathon and early usage.

---

## 14. Deployment & Environment

**Server:** DigitalOcean 1GB droplet, Ubuntu, nginx reverse proxy with Certbot SSL at `wahwa.com`

**Backend startup:**
```bash
source ~/miniconda3/bin/activate && conda activate lockinbro
uvicorn app.main:app --host 0.0.0.0 --port 3000 --reload
```

| Setting | Value |
|---------|-------|
| Python | 3.12.13 (conda) |
| FastAPI root_path | `/api/v1` |
| Port | 3000 |
| Database | PostgreSQL `focusapp` on localhost:5432 |
| SSL | nginx + Certbot (not handled by FastAPI) |
| Claude model | claude-sonnet-4-20250514 |
| JWT access token TTL | 60 min |
| JWT refresh token TTL | 30 days |
| Password hashing | Argon2id |
| DB pool | min=2, max=10 (asyncpg) |

**Required `.env` variables:**
```
DATABASE_URL=postgresql://root@localhost:5432/focusapp
JWT_SECRET=<secret>
ANTHROPIC_API_KEY=<key>
HEX_API_TOKEN=<token>
HEX_NB_DISTRACTIONS=<notebook_id>
HEX_NB_FOCUS_TRENDS=<notebook_id>
HEX_NB_WEEKLY_REPORT=<notebook_id>
APPLE_BUNDLE_ID=com.lockinbro.app
```

**Key dependencies:** FastAPI 0.135.2, SQLAlchemy 2.0.48, asyncpg 0.31.0, anthropic 0.86.0, argon2-cffi 25.1.0, python-jose 3.5.0, httpx 0.28.1

---

## 15. Implementation Status & Remaining Gaps

### Backend (lockinbro-api/) — Implemented

All core endpoints, schemas, services, and migrations are complete. The following is fully functional:

| Component | Status | Notes |
|-----------|--------|-------|
| Auth (register, login, Apple, refresh) | Done | Apple token verification skips signature check (hackathon) |
| Tasks (CRUD, brain-dump, plan) | Done | Includes GET /tasks/upcoming |
| Steps (list, update, complete) | Done | |
| Sessions (start, join, checkpoint, end, resume) | Done | Enforces one active session per user |
| Distractions (screenshot VLM, app-check, app-activity) | Done | VLM auto-updates steps server-side |
| Analytics (Hex notebooks + direct DB summary) | Done | Returns raw Hex responses |
| LLM service (6 prompt functions) | Done | Includes suggest_work_apps for handoff |
| Push service (APNs) | **Stub** | Logs only — see gap #1 below |
| Database migrations | Done | 2 migrations (initial + cross-device) |

### Remaining Gaps (Must Implement)

**Gap 1: APNs push notifications (`services/push.py`)**

`send_push()` and `send_activity_update()` are stubs that only log. Need real HTTP/2 APNs calls. Required for:
- Cross-device session handoff notifications
- ActivityKit Live Activity start/update/dismiss on iPad
- Smart notifications (deadline approaching, morning brief, etc.)

Requires:
- APNs `.p8` auth key from Apple Developer account
- `APNS_KEY_ID`, `APNS_TEAM_ID`, `APNS_KEY_PATH` env vars
- HTTP/2 client (httpx supports this) to `api.push.apple.com`
- Two push types: `alert` (notifications) and `liveactivity` (ActivityKit)

**Gap 2: Device token registration endpoint**

`push.register_device_token()` exists in services but is **not wired to any router**. Need:

```
POST /auth/device-token
Authorization: Bearer <jwt>
{
  "platform": "mac | ipad | iphone",
  "token": "APNs device token hex string"
}
```

Each client calls this on app launch and whenever the APNs token refreshes. Upserts into `users.device_tokens` JSONB.

**Gap 3: Apple identity token verification (`middleware/auth.py`)**

Currently decodes Apple's JWT without signature verification (line 85, marked as hackathon shortcut). Production requires:
- Fetch Apple's JWKS from `https://appleid.apple.com/auth/keys`
- Verify RS256 signature, issuer (`https://appleid.apple.com`), audience (bundle ID)

**Gap 4: `prioritize_tasks()` — no endpoint**

`llm.prioritize_tasks()` is implemented in `services/llm.py` but not called by any router. If needed, wire to `POST /tasks/prioritize`.

**Gap 5: `distraction_patterns` table — never populated**

Schema exists (migration 001) but no code writes to it. Could be a background job that aggregates from the `distractions` table, or populated during Hex notebook runs.

---

## 16. Key Technical Notes

- **1GB RAM constraint:** FastAPI + uvicorn ≈ 40–60MB. Postgres already running. Screenshots received as raw JPEG binary (~40–60 KB each at quality 0.5) are transient — GC'd quickly. Stay mindful of concurrent VLM requests (limit to 1–2 in-flight at a time).
- **Claude Code:** Run from local machine via SSH (`ssh -t` or VS Code Remote), not installed on the droplet. Saves RAM and disk.
- **VLM prompt context:** The screenshot analysis prompt includes full task context (title, goal, all steps with statuses and checkpoint_notes) so Claude can determine exactly what the user is working on and write accurate checkpoint_note updates.
- **Step auto-update is a backend side-effect:** When `/distractions/analyze-screenshot` returns, the backend applies step status changes and checkpoint_note updates before returning the response to the client. The client doesn't need to make separate step-update calls.
- **Auth already implemented:** Backend uses Argon2id hashing + HS256 JWT tokens. Apple Sign In token verification is stubbed for hackathon (decodes without signature check).
