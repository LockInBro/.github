# LockInBro — Cognitive Prosthetic for the ADHD Brain

## Technical Design Document — YHack 2026 Edition
**v3.0 · March 28–29, 2026 · Yale University, New Haven CT**
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

**Core thesis:** LockInBro is not a productivity tool. It's an external executive function system — the cognitive infrastructure that ADHD brains are missing. The agent perceives your environment (VLM), decides what you should be doing (task prioritization), remembers where you left off (checkpoint notes), detects when you're stuck in a friction loop (proactive pattern detection), and intervenes by DOING the work for you (not just nudging). Each feature replaces a specific executive function deficit: working memory, task initiation, task switching, time perception, and impulse control.

**Why this matters:** 366 million adults worldwide have ADHD. Their brains' executive function systems work differently, making it hard to start tasks, stay on them, manage time, and recover from interruptions. Almost no software is built for cognitive accessibility — this is one of the largest underserved populations in tech.

---

## 1.5. The Proactive Agent Philosophy (Argus Layer)

Traditional AI is reactive: it waits in a chat box for you to articulate your problem. LockInBro's Argus layer shifts the paradigm. It runs in the background, continuously analyzing your screen state over time to detect **friction patterns** — not just "are you distracted?" but "are you struggling with something the AI could handle?"

### Core Proactive Behaviors

**Behavior 1: Repetitive Loop Detection → Auto-Extract**
The VLM receives a rolling buffer of recent screenshots (not just the current one). When it detects the user switching between the same 2-3 windows repeatedly (image → Calendar, email → spreadsheet, PDF → form), it identifies the pattern and offers to do the work:
- "I see you're copying events from this schedule image to Calendar. Want me to extract all 14 and populate them?"
- "You're pulling numbers from this PDF into the spreadsheet. Want me to fill the remaining cells?"
- User approves → system executes via AppleScript (fast, native) or Claude Computer Use (general, slower).

**Behavior 2: Gaze-Driven Reading Assistant (toggleable)**
Eye tracking watches where the user is reading on screen. When the user re-reads the same line 3+ times, or stalls on a paragraph for 2x their normal reading speed, the system infers confusion at that specific spot. Without the user asking, an explanation card appears anchored next to where their eyes stalled — defining terms, simplifying concepts, or connecting to earlier material. Your eyes are the query. This is toggleable in settings (off by default, turn on in Settings or via menu bar).

**Behavior 3: Proactive Context Resume**
When the Argus layer detects the user returning to a previously-active task (e.g., reopening VS Code after 30 minutes away, or switching back to a document), it automatically surfaces the context checkpoint — "You were on 'Write methods section', got through intro and background paragraphs, results paragraph is next." No button press needed; the system recognizes the task resumption from the screen state.

**Behavior 4: Intent Disambiguation**
When the system detects the user looking at content that could mean different things, it offers MULTIPLE action options instead of guessing:
- User is looking at a Canvas notification → system offers:
  - "Add this to your task list?" (skimming mode)
  - "Help you complete this assignment?" (doing mode)
  - "Dismiss" (not relevant)
- The system learns from past choices. After 5+ interactions where the user always picks "add to tasks" for Canvas notifications, it defaults to that and just asks for confirmation.

### Screenshot History Buffer (Updated from Implementation)

The proactive detection system requires temporal context — a single screenshot can't detect "user is in a loop." The system maintains a **two-tier rolling buffer**:

- **Batched capture:** Screenshots captured every **2.5 seconds**, but VLM is called only every **4th capture** (~10 seconds). This gives the VLM 4 frames spanning 10 seconds of activity per call — enough temporal context for diff analysis while keeping API costs manageable.
- **Image tier:** `deque(maxlen=4)` of recent screenshots, ALL sent as images. The VLM visually diffs between frames — detecting cursor movement, new text, scroll changes, window switches. Text summaries lose the fine-grained detail needed for temporal analysis.
- **Text tier:** `deque(maxlen=12)` of older VLM summaries (~60s of history). When images roll off the image buffer, their text summaries persist here for extended context (e.g., "user read a problem description 60s ago").
- **Previous output:** The VLM's last JSON output is fed back into the next prompt for self-refinement. This lets the model correct or build on its previous analysis rather than guessing from scratch.
- **Execution context:** After the agentic executor completes an action, its summary is injected into the VLM prompt so the model knows the task was handled and can look for what the user does next.
- **Buffer structure:** `HistoryBuffer(image_maxlen=4, text_maxlen=12)` with `_last_output` and `_last_execution` fields

### Proactive Action Execution (Updated from Implementation)

When friction is detected and the VLM includes `proposed_actions`, the system:

1. **Deduplicates notifications and filters by action type.** A `NotificationManager` tracks the fingerprint (friction type + action labels) of the last shown card. Only shows a new card when the proposed action meaningfully changes — prevents spam. Additionally, only actions with executor-actionable types (`auto_extract`, `auto_fill`, `summarize`, `brain_dump`) trigger friction cards. Generic suggestions (`action_type: "other"`) fall through to the gentle nudge path instead.

2. **Shows a non-intrusive action card:**
   ```
   ┌──────────────────────────────────────────────┐
   │ 🔧 Friction detected                         │
   │                                               │
   │ You're manually transcribing from a receipt.  │
   │                                               │
   │  [1] Extract receipt data to lunch.md         │
   │  [0] Not now                                  │
   └──────────────────────────────────────────────┘
   ```

3. **On approval, an agentic executor runs.** Not a single-shot LLM call — a full agent loop (Gemini 3 Flash) with tool use:
   - **Tools available:** `read_file`, `write_file` (existing text files only), `output` (display to user), `run_command`, `done`
   - **Vision:** Executor receives all buffered screenshots so it can read source content (receipts, PDFs, code) directly from the images
   - **Agent loop:** Gemini proposes tool calls → tools execute → results fed back → repeat (up to 10 steps)
   - **File safety:** `write_file` only works on existing plain text files with a confirmed path. Binary format targets (docx, ppt, forms) use `output()` — user copies from the displayed result.
   - **File discovery:** Agent uses `mdfind` (macOS Spotlight) or `lsof` to locate files by name when only the filename is visible on screen.

4. **Output routing:**
   - **Existing text files (code, markdown, config):** Agent writes directly via `write_file` after confirming path with `read_file`
   - **Binary/external targets (docx, ppt, website forms):** Agent uses `output()` to display content in a sticky-note UI. User copies.
   - In production (Swift app), `output()` becomes a floating card / pasteboard copy.

5. **Execution results fed back to VLM.** After the executor completes, its summary is injected into the VLM prompt so the model knows the task was handled and stops re-flagging the same friction.

6. **Learns from preferences:** Store user choices in a `user_preferences` JSONB field. After 5+ consistent choices for the same pattern type, default to that action and just ask for one-tap confirmation.

### Upgraded VLM Prompt (Updated from Implementation)

The VLM analyzes a **time sequence of screenshots** (not a single frame). The primary signal is the **diff between consecutive frames** — where pixels changed = where the user's attention is. Static content is background noise.

Key prompt design principles (learned from iteration):
- **Diff-first analysis.** The prompt explicitly tells the model to compare frames and focus on what changed. An error that was ALREADY THERE in all frames is stale; an error that APPEARED between frames is relevant.
- **Task inference from screen.** If no explicit task is set, the VLM infers the task from what the user is actively doing (where pixels change), not from static content.
- **Fast friction detection.** Each frame is 4s apart. 2 unchanged frames = 8s idle = significant. Write-then-delete = stuck immediately. Repeated source→target switching = tedious_manual on the second switch.
- **Rich proposed_actions.** The `details` field is a natural language spec for the agentic executor: what the user is trying to do, where to look in the screenshots, what format/style to match. The executor has vision too — tell it WHERE to look and WHAT to do, not the raw data.
- **Session awareness.** The prompt includes open sessions from the backend so the VLM can detect task resumption and suggest session actions (resume, switch, complete, start new).
- **Self-refinement.** Previous VLM output + execution results are fed back into the next prompt.

VLM JSON output schema:
```json
{
  "on_task": true,
  "current_step_id": "step UUID or null",
  "inferred_task": "what the user is actually working on, based on screen diffs",
  "checkpoint_note_update": "what changed across these frames specifically",
  "steps_completed": ["UUIDs"],
  "friction": {
    "type": "repetitive_loop | stalled | tedious_manual | context_overhead | task_resumption | none",
    "confidence": 0.0-1.0,
    "description": "what the user is struggling with, based on diff evidence",
    "proposed_actions": [
      {"label": "specific verb phrase", "action_type": "auto_extract | brain_dump | auto_fill | summarize | other", "details": "natural language spec for executor: what to do, where to look, what format"}
    ],
    "source_context": "filename or app name",
    "target_context": "filename or app name"
  },
  "session_action": {
    "type": "resume | switch | complete | start_new | none",
    "session_id": "uuid or null",
    "reason": "why this session action is suggested"
  },
  "intent": "skimming | engaged | unclear | null",
  "distraction_type": "app_switch | browsing | idle | null",
  "app_name": "primary visible application",
  "confidence": 0.0-1.0,
  "gentle_nudge": "nudge text if distracted, null otherwise",
  "vlm_summary": "1-sentence description of what CHANGED across frames"
}
```

VLM model: **Gemini 2.5 Pro** — strong instruction following, handles complex multi-rule prompts reliably. Called every ~10s with 4 frames.
Executor model: **Gemini 3 Flash** (preview) — fast, agentic tool use for action execution with 429 retry logic.

### Notification Priority (Friction Help > Nudge)

When the system detects something, it prioritizes in this order:

1. **Proactive friction help** (blue card, "I can do this for you") — if the system can take a concrete action
2. **Context resume** (warm card, "Welcome back, here's where you left off") — if task resumption detected
3. **Gentle drift nudge** (amber, "Hey, you were working on...") — ONLY if no actionable help applies

The system should NEVER nudge when it could help instead. Helping > nagging.

### New DB Table: Proactive Actions Log

```sql
CREATE TABLE public.proactive_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES public.sessions(id) ON DELETE SET NULL,
    friction_type   TEXT NOT NULL,          -- repetitive_loop | stalled | tedious_manual | etc.
    proposed_action TEXT NOT NULL,
    user_choice     TEXT,                   -- accepted | declined | alternative_chosen
    chosen_action   TEXT,                   -- what the user actually picked (if multi-option)
    executed        BOOLEAN DEFAULT false,
    detected_at     TIMESTAMPTZ DEFAULT now(),
    responded_at    TIMESTAMPTZ
);

CREATE INDEX idx_proactive_user ON proactive_actions(user_id, friction_type);
```

This table feeds into preference learning: query past choices by `friction_type` to determine default actions.

### New API Endpoints for Proactive Actions

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/proactive/respond` | User responds to a proactive action card (accepted/declined/alternative). Logs choice, triggers execution if accepted. |
| POST | `/proactive/execute` | Execute an approved proactive action (AppleScript, Computer Use, or API). |
| GET | `/proactive/preferences` | Get learned preferences for action defaults (used by macOS client to pre-select actions). |

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

**Flow 1: Real-time Friction Detection + Distraction Detection (macOS, device-side VLM)**
- ScreenCaptureKit captures a screenshot every **2.5 seconds**. VLM is called every **4th capture** (~10 seconds) with all 4 frames batched together.
- **VLM runs device-side:** Mac calls Gemini 2.5 Pro API with **4 screenshots as images** spanning ~10 seconds, plus text summaries of older analyses (12-entry text history). The VLM analyzes the **diff between consecutive frames** to detect user attention, friction patterns, and task state.
- VLM returns enriched JSON: inferred task, step updates, friction detection, session actions, and proposed proactive actions with natural language specs for the executor
- Mac sends **only the JSON result** (no image) to `POST /distractions/analyze-result` on the backend
- Backend applies side-effects: updates step statuses + `checkpoint_note`, logs distractions, stores proactive actions for preference learning
- If friction detected with proposed actions: Mac shows action card. User approves → **agentic executor** (Gemini 3 Flash) runs with tool use (read_file, write_file, output, run_command) up to 10 steps. Executor receives buffered screenshots for vision context.
- If distraction detected: Mac shows gentle nudge
- If session action detected (resume/switch/complete): Mac shows session card
- Screenshots discarded from device memory after VLM response (never persisted, never transit through backend)

**Flow 2: Brain-Dump Task Parsing (iOS, async)**
- User speaks or types a stream-of-consciousness dump
- If voice: on-device Speech framework transcribes to text
- Text sent to `POST /tasks/brain-dump` → Claude extracts structured tasks
- Returns parsed tasks with inferred deadlines, priorities, time estimates
- App asks "want me to break this into steps?" → `POST /tasks/{id}/plan` → Claude generates 5–15 min steps
- User reviews, edits if needed, confirms

**Flow 3: Focus Session + Auto-Update (macOS, VLM-driven lifecycle)**
- **Session start:** VLM runs in always-on mode. On startup (and periodically), fetches `GET /sessions/open` to get all active + interrupted sessions. When VLM detects the user working on something matching an existing task (same app + file), it suggests starting/resuming a session via `session_action` output. User approves → `POST /sessions/start` → returns full task context.
- **During session:** VLM gets full task/step context in prompt. Auto-updates step statuses + checkpoint_note via `POST /distractions/analyze-result`.
- **Session resume:** When VLM detects user returning to a previously-interrupted session (screen matches interrupted session's last app/file + time away > 5 min), it outputs `session_action: resume`. App calls `GET /sessions/{id}/resume` → shows context card: "You were on 'Write methods section' — got through 3 of 5 paragraphs. Results next."
- **Session switch:** When VLM detects user moved to a different task matching another open session, it suggests switching.
- **Session complete:** When VLM detects user finished (all steps done, or user moved to unrelated work for extended time), it suggests completing the session → `POST /sessions/{id}/end`.
- On interruption, checkpoint saved → `POST /sessions/{id}/checkpoint`

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
| Screenshots (macOS only) | NEVER persisted. Analyzed device-side via VLM API call (Gemini/Claude). Only the JSON analysis result is sent to the backend — screenshots never transit through the backend server. | TLS to VLM API only |
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
| PATCH | `/steps/{id}` | Update step fields (title, description, estimated_minutes, sort_order, status, checkpoint_note) |
| POST | `/steps/{id}/complete` | Mark step as done with completed_at timestamp |
| DELETE | `/steps/{id}` | Hard-delete a step (ownership verified via parent task) |

### Sessions

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/sessions/active` | Get the current active session for the authenticated user. Returns 404 if none. |
| GET | `/sessions/open` | **New.** All active + interrupted sessions for the user. Used by VLM on startup to enable session-aware analysis (task resumption detection, session switching). |
| POST | `/sessions/start` | Start focus session, optionally linked to a task. Returns full task context + all steps. Can be triggered by VLM suggestion (user approves "Start focus session?" card). |
| POST | `/sessions/{id}/checkpoint` | Save context checkpoint (current step, last action, attention score, active app) |
| POST | `/sessions/{id}/end` | End session with status (completed / interrupted). Can be triggered by VLM suggestion (user approves "Complete session?" card). |
| POST | `/sessions/{id}/join` | Second device joins an active session. Returns full task context + suggested work app. Avoids duplicate sessions. |
| GET | `/sessions/{id}/resume` | Get last checkpoint + step checkpoint_notes → AI-generated resume card |

**VLM-driven session lifecycle:** The VLM operates in two modes:
1. **Always-on (no session):** Runs at low cadence, infers task from screen. Can suggest starting a session when it detects the user working on something that matches an existing task.
2. **Session mode:** VLM gets full task/step context from the session, sends results to backend via `POST /distractions/analyze-result`. Can suggest completing a session when it detects the user finished or moved on.

The VLM fetches `GET /sessions/open` on startup and periodically, injecting all open sessions into its prompt. When it detects a screen match (same app + same file as an interrupted session), it outputs `session_action: {type: "resume", session_id: "..."}` and the app shows a resume card.

### Distraction Detection

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/distractions/analyze-result` | **Primary.** Desktop sends pre-analyzed VLM JSON result (no image). Backend applies side-effects: step updates, distraction logging, proactive action storage. Device-side VLM architecture — screenshots never reach the backend. |
| POST | `/distractions/analyze-screenshot` | **Legacy fallback.** Desktop sends screenshot (raw JPEG binary via multipart) + task context. Backend runs VLM. For testing or when device-side VLM is unavailable. |
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

### POST /distractions/analyze-result (Primary — Device-Side VLM)

The macOS app runs VLM locally (calls Gemini/Claude Vision API directly), then sends **only the JSON result** to the backend. No image ever reaches the server.

**Request:**
```json
{
  "session_id": "uuid",
  "on_task": false,
  "current_step_id": "uuid-of-step-2",
  "checkpoint_note_update": "wrote intro and background paragraphs, results paragraph is next",
  "steps_completed": ["uuid-of-step-1"],
  "friction": {
    "type": "none",
    "confidence": 0.0,
    "description": null,
    "proposed_actions": [],
    "source_context": null,
    "target_context": null
  },
  "intent": null,
  "distraction_type": "browsing",
  "app_name": "Safari",
  "confidence": 0.92,
  "gentle_nudge": "Looks like Twitter pulled you in! You were making great progress on the methods section — just the results paragraph left.",
  "vlm_summary": "User has Safari open with Twitter feed visible. VS Code in background with presentation file."
}
```

**Response (200):**
```json
{
  "side_effects_applied": true,
  "steps_updated": 2,
  "distraction_logged": true,
  "proactive_action_id": null
}
```

**Backend side-effects:**
1. `UPDATE steps SET status = 'done', completed_at = now() WHERE id IN (completed step UUIDs)`
2. `UPDATE steps SET checkpoint_note = '...', last_checked_at = now() WHERE id = current_step_id`
3. `INSERT INTO distractions (...)` if `on_task = false`
4. `INSERT INTO proactive_actions (...)` if `friction.type != "none"` and `friction.confidence > 0.7`
5. Update session checkpoint with latest analysis

### POST /distractions/analyze-screenshot (Legacy Fallback)

For testing or when device-side VLM is unavailable. Backend runs VLM server-side.

**Request:** multipart/form-data
```
screenshot: raw JPEG binary (UploadFile, 1280x720, quality 0.5)
window_title: string
session_id: UUID
task_context: JSON string (same shape as before)
```

**Response:** Same JSON shape as analyze-result request body (the VLM output).

**Backend side-effects:** Same as analyze-result.

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

### GET /sessions/active

**Request:** None (authenticated via JWT).

**Response (200):** Same `SessionOut` shape as start:
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

**404** if no active session exists. Used by the frontend to recover a stale session ID after app restart — the client can then call `POST /sessions/{id}/end` to clear it before starting a new session.

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
1. Enforces one active session per user — returns 409 with `{"message": "...", "active_session_id": "uuid"}` if one already exists
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
  "title": "Updated step title",
  "description": "Updated description",
  "estimated_minutes": 15,
  "sort_order": 3,
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

### DELETE /steps/{id}

**Request:** None (empty body).

**Response:** 204 No Content.

Hard-deletes a step. Ownership is verified by joining through the parent task's `user_id`. Returns 404 if the step doesn't exist or belongs to another user.

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

LockInBro registers the URL scheme `lockinbro://` for cross-device deep links:

| URL | Triggered By | Action |
|-----|-------------|--------|
| `lockinbro://join-session?id={session_id}&open={url_encoded_app_scheme}` | Live Activity tap / Push notification tap | Join session + chain-open target app |
| `lockinbro://resume-session?id={session_id}` | Push notification (session resume) | Open resume card for session |
| `lockinbro://task?id={task_id}` | Push notification (deadline, morning brief) | Open task detail view |

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
| Screenshot Engine | ScreenCaptureKit | Periodic screen capture every 5s during active sessions. Resolution: 1280x720, JPEG quality 0.5. Analyzed device-side via VLM API — only JSON results sent to backend. |
| VLM Client | Gemini/Claude SDK | Device-side VLM calls. Manages rolling `deque(maxlen=4)` of recent analyses. Sends current screenshot + text summaries of previous 3-4 analyses to VLM API. |
| Proactive Action Cards | SwiftUI | Floating UI elements for friction detection actions. Shows near relevant screen region. Multiple action options for ambiguous intent. |
| Gaze Tracker | AVFoundation + CoreML | L2CS-Net model converted to CoreML. Tracks where user is looking on screen. |
| Process Monitor | NSWorkspace | Observes frontmost app changes. Auto-detects work apps and suggests starting a focus session. |
| Notification Engine | UserNotifications | Delivers distraction alerts with gentle nudges, context reminders, and session prompts. |

### Friction Detection Pipeline (Device-Side VLM)

1. **Trigger:** Timer fires every 5 seconds during active focus session
2. **Capture:** ScreenCaptureKit captures primary display at 1280x720, JPEG quality 0.5
3. **Pre-filter:** Compare with previous screenshot using perceptual hash. If >95% similar, skip VLM call to save cost.
4. **Analyze (device-side):** Mac calls Gemini/Claude Vision API directly. Sends current screenshot as image + text summaries of last 3-4 VLM analyses from the rolling `deque(maxlen=4)`. VLM returns enriched JSON with task status, friction detection, intent, and proposed actions.
5. **Store summary:** Append VLM summary to rolling deque (text only, ~50 tokens). Discard screenshot from memory immediately.
6. **Send to backend:** `POST /distractions/analyze-result` with only the JSON result (no image). Backend applies DB side-effects (step updates, distraction logs, proactive action storage).
7. **Act locally:**
   - Friction detected (confidence > 0.7) → show proactive action card near relevant screen region
   - Distraction detected (confidence > 0.7) → fire gentle nudge notification
   - Task resumption detected → auto-surface context resume card
   - Confidence 0.5–0.7 → log but don't interrupt

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
| User left LockInBro | UIApplication lifecycle | `sceneDidEnterBackground` starts a local timer. If user doesn't return within threshold, schedule nudge notification. |
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
| Tap behavior | Opens LockInBro → chains to target app | Same — but persistent visibility is the key difference |

A focus session IS an ongoing activity — exactly what Live Activities were designed for.

#### Handoff Flow

1. **Mac starts session** → `POST /sessions/start` with `platform: "mac"`
2. **Backend sends two things to iPad via APNs:**
   - **Push notification** (initial alert): "Focus session active: 'Take notes on Q1 report'. Tap to join on iPad." **[Join on iPad]** — **[Dismiss]**
   - **ActivityKit push** (starts Live Activity): persistent lock screen widget showing task title + current step + "Open Notes" — stays visible for the entire session
3. **Mac advertises `NSUserActivity`** with task context — iPad shows Handoff icon on lock screen / dock as a tertiary entry point
4. **User picks up iPad** → sees Live Activity on lock screen (or the push notification, or Handoff icon) → **taps once**
5. **Deep link chain fires** — LockInBro opens via deep link → `onOpenURL` handler immediately calls `UIApplication.shared.open(targetURL)` **before UI renders** → target app (Notes, Pages, etc.) opens. User sees: tap → Notes opens. LockInBro flashes for a fraction of a second or not at all.
6. **LockInBro joins session in background** — `POST /sessions/{id}/join` called during the deep link handler. DeviceActivityMonitor configured with work app whitelist. Live Activity continues showing on lock screen.
7. **Both devices now feed into the same session** — Mac VLM updates steps, iPad tracks app usage + manual step completions. All writes go to the same `steps` table.

#### The Deep Link Chain (Key Implementation Detail)

The trick that makes this feel seamless: handle the deep link and chain-open the target app **before your own UI renders.** The user taps the Live Activity → Notes opens. LockInBro is just a passthrough.

```swift
// LockInBro deep link handler — fires before UI renders
.onOpenURL { url in
    // url = lockinbro://join-session?id=xxx&open=mobilenotes%3A%2F%2F
    if let targetScheme = url.queryParam("open"),
       let targetURL = URL(string: targetScheme) {
        // Chain-open immediately — target app opens before LockInBro UI appears
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
let activity = NSUserActivity(activityType: "com.lockinbro.focus-session")
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
lockinbro/
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
├── LockInBro/                           # iOS + iPadOS — single Xcode project, 4 targets
│   │                                    # Separate repo/project from macOS (different developer)
│   ├── LockInBro/                       # Target 1: Main app (universal: iPhone + iPad)
│   │   ├── LockInBroApp.swift
│   │   ├── DeepLinkHandler.swift        # onOpenURL → chain-open target app for cross-device handoff
│   │   ├── Views/
│   │   │   ├── BrainDumpView.swift
│   │   │   ├── TaskBoardView.swift
│   │   │   ├── DashboardView.swift      # Hex analytics display
│   │   │   ├── SettingsView.swift
│   │   │   └── iPad/                    # iPad-only views (gated on UIDevice.userInterfaceIdiom == .pad)
│   │   │       ├── iPadFocusSessionView.swift
│   │   │       └── WorkAppPickerView.swift
│   │   ├── Models/                      # Task, Step, Session, etc. (Codable structs)
│   │   └── Services/
│   │       ├── APIClient.swift
│   │       ├── SpeechService.swift
│   │       ├── ScreenTimeManager.swift
│   │       ├── DeviceActivityManager.swift  # iPad: configures the monitor extension
│   │       └── HandoffReceiver.swift        # iPad: handles NSUserActivity from Mac + session join
│   ├── LockInBroWidgets/                # Target 2: Widget Extension (Live Activity)
│   │   ├── FocusSessionLiveActivity.swift   # Lock screen widget UI (step progress, "Open Notes" / "Mark Done")
│   │   └── FocusSessionAttributes.swift     # ActivityKit attributes (shared with main app via App Group)
│   ├── LockInBroMonitor/                # Target 3: DeviceActivityMonitor Extension
│   │   └── MonitorExtension.swift       # Fires when off-task app usage exceeds threshold → local notification
│   └── LockInBroShield/                 # Target 4: ShieldConfiguration Extension
│       └── ShieldProvider.swift         # Custom shield UI ("Hey, quick check-in!")
│   # All 4 targets share an App Group (group.com.lockinbro.app) for cross-process data
│   # Capabilities: FamilyControls, Push Notifications, App Groups on main target
│
└── LockInBroMac/                        # macOS — separate Xcode project (different developer)
    ├── LockInBroMacApp.swift
    ├── Models/                          # Copy of Task, Step, Session structs (same as iOS)
    ├── Networking/                      # Copy of APIClient (same backend, same contracts)
    ├── MenuBar/
    │   └── StatusBarController.swift
    ├── FocusSession/
    │   ├── SessionWindow.swift
    │   ├── ScreenshotEngine.swift
    │   ├── ContextCheckpoint.swift
    │   └── HandoffAdvertiser.swift      # Publishes NSUserActivity for cross-device handoff
    ├── Argus/                           # Device-side VLM + friction detection
    │   ├── VLMClient.swift              # Gemini/Claude Vision API calls (device-side)
    │   ├── AnalysisBuffer.swift         # Rolling deque(maxlen=4) of recent VLM summaries
    │   ├── FrictionDetector.swift       # Interprets VLM output, triggers action cards
    │   └── ProactiveActionCard.swift    # SwiftUI floating action card UI
    ├── EyeTracking/
    │   ├── GazeTracker.swift
    │   └── L2CSNet.mlpackage
    └── ProcessMonitor/
        └── AppWatcher.swift
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

1. **Hour 0–1:** Postgres schema + migrations (including `proactive_actions` table), auth endpoints, FastAPI scaffold
2. **Hour 1–3:** `POST /tasks/brain-dump` + `POST /tasks/{id}/plan` + `GET /tasks` + `GET /tasks/{id}/steps` — brain dump is the entry point
3. **Hour 3–6:** Screenshot analysis pipeline WITH history buffer (5s interval, 4-frame window) + upgraded friction detection VLM prompt + macOS screenshot engine
4. **Hour 6–9:** ★ **PROACTIVE ACTION SYSTEM** — action card UI on macOS + `/proactive/respond` + `/proactive/execute` + AppleScript templates for Calendar auto-fill, email drafting, form filling. **This is the demo-defining feature. Get it working by hour 9.**
5. **Hour 9–12:** Context resume as proactive behavior (auto-detect task resumption from screen state) + `GET /sessions/{id}/resume` + intent disambiguation (multi-option cards)
6. **Hour 12–15:** iOS brain dump view + task board + Hex setup + analytics notebooks
7. **Hour 15–18:** Integration testing, end-to-end demo flow rehearsal, preference learning from `proactive_actions` table
8. **Hour 18–21:** Gaze reading assistant (toggleable, stretch goal), UI polish, edge case fixes
9. **Hour 21–23:** Record demo video, submit to Devpost, rehearse pitch

### Demo Script (3-minute pitch)

1. **[0:00–0:30] Problem + Stats:** "366M adults with ADHD are missing neural hardware. We built the software replacement. LockInBro is a cognitive prosthetic — it perceives your environment, remembers where you left off, detects when you're stuck, and does the work for you." *(ASUS Healthcare + Harper AI Agent setup)*
2. **[0:30–1:00] Brain Dump → Steps (live):** Speak a messy thought stream into iPhone. Show Claude parsing into tasks, then breaking a task into 5–15 min steps. "This replaces task initiation — the executive function that makes starting feel impossible." *(Harper track: AI agent)*
3. **[1:00–1:45] ★ PROACTIVE MOMENT — Friction Detection (live, the WOW):**: Start a focus session. Have a JPEG schedule image open next to Apple Calendar. Manually copy 2-3 events. On the 3rd copy, system detects the repetitive loop → action card appears: "I noticed you're copying events from this image to Calendar. Want me to extract all 14?" Click approve. Watch the system auto-populate Calendar. Hands off keyboard. "This replaces the tedious manual work that burns ADHD brains out." *(Harper track: proactive autonomous agent — this is the demo-defining moment)*
4. **[1:45–2:10] Proactive Context Resume:** Switch away from the task for 30 seconds. Come back to the same app. System AUTOMATICALLY surfaces the context card: "You were on 'Write methods section' — got through the intro and background. Pick up at results." No button pressed. It detected task resumption from the screen state. "This replaces working memory — the executive function that forgets where you were." *(Harper track)*
5. **[2:10–2:35] Analytics Dashboard:** Show the Hex-powered dashboard: distraction patterns by hour, top distracting apps, focus trend chart, friction events detected. *(Hex track)*
6. **[2:35–3:00] Gaze Reading Assistant (if working) + Close:** Briefly show gaze-anchored explanation card appearing where eyes stalled on a document. "Your eyes are the query." Close: "This is a cognitive prosthetic. Every feature replaces a specific deficit. LockInBro doesn't watch you. It thinks for you."

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

- **1GB RAM constraint:** FastAPI + uvicorn ≈ 40–60MB. Postgres already running. With device-side VLM, the backend never handles screenshots — only receives small JSON results (~1KB each). No VLM memory pressure on the droplet.
- **Device-side VLM architecture:** Screenshots are captured and analyzed on the Mac via direct Gemini/Claude API calls. Backend receives only pre-analyzed JSON via `POST /distractions/analyze-result`. This eliminates image upload bandwidth, reduces latency (one fewer hop), strengthens privacy (screenshots never transit through backend), and keeps the droplet RAM-safe. The legacy `analyze-screenshot` endpoint remains as a fallback for testing.
- **Claude Code:** Run from local machine via SSH (`ssh -t` or VS Code Remote), not installed on the droplet. Saves RAM and disk.
- **VLM prompt context:** The upgraded Argus prompt includes task context + rolling history of recent analyses (text summaries, not images). The Mac manages the `deque(maxlen=4)` buffer locally.
- **Step auto-update is a backend side-effect:** When `/distractions/analyze-result` receives VLM output, the backend applies step status changes, checkpoint_note updates, and proactive action logging before responding.
- **Auth already implemented:** Backend uses Argon2id hashing + HS256 JWT tokens. Apple Sign In token verification is stubbed for hackathon (decodes without signature check).
