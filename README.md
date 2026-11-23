# 3sReach

**Win the first 3 seconds. Grow your reach.**

Score a short video before you upload it to TikTok, Instagram, YouTube or Facebook.
*(Pronounced "three-second reach". In advertising, **reach** is the number of people who see your content.)*

Upload a video and 3sReach checks it against each platform's format guidelines, uses AI to judge
the hook, pacing and clarity, and gives a 1–10 score per platform with concrete tips. You can also compare
two versions of a video (A/B) and see which one is likely to perform better.

> ⚠️ **What this does not do:** nobody can predict an exact view count, not even the platforms
> themselves. Views also depend on the account's audience, timing, trends and the algorithm.
> 3sReach predicts **relative creative quality**: "version B is stronger than version A, and here's why".

Most of the project is written in strict **TypeScript** (a pure scoring library, a NestJS API and a
React frontend), with a **Python** service for video processing and the AI review.

---

## Table of contents

1. [Features](#features)
2. [Tech stack](#tech-stack)
3. [Architecture](#architecture)
4. [How a video is scored](#how-a-video-is-scored)
5. [Project structure](#project-structure)
6. [Milestones: build it step by step](#milestones-build-it-step-by-step)
7. [Core scoring functions](#core-scoring-functions)
8. [Data model](#data-model)
9. [API](#api)
10. [Configuration](#configuration)
11. [Running locally](#running-locally)

---

## Features

**Version 1 (MVP)**
- Upload a video (MP4/MOV, up to ~200 MB)
- Automatic checks: aspect ratio, resolution, length, frame rate, audio present, loudness, captions
- AI review of key frames and the spoken transcript: hook strength (first 3 seconds), clarity, pacing, call-to-action
- A score from 1 to 10 for **TikTok, Instagram Reels, YouTube Shorts and Facebook Reels**, each with tips
- A history of analysed videos

**Version 2**
- **Frame timeline**: the video's key frames in a strip; click a frame to see what the AI said about that moment
- **Score charts** for the four platforms
- A/B comparison: upload two versions, get a side-by-side verdict
- Live progress while a video is analysed

**Version 3 (stretch)**
- Learn from real results: enter the actual views a video got after upload, and calibrate the scores to your own audience
- Deploy to Kubernetes

---

## Tech stack

| Layer | Technology | Why |
|---|---|---|
| Language | **TypeScript** in `strict` mode everywhere except the analyzer | Catches bugs at compile time; strict mode is how professional teams use it |
| Scoring library | **Pure TypeScript** package, no framework | Rules and scoring logic as pure functions: easy to unit test and reuse |
| Frontend | **React + TypeScript** (Vite), **React Router**, **TanStack Query** for server data, **Zustand** (or `useReducer`) for UI state | Industry-standard frontend stack; server state and UI state are kept separate |
| Charts | **Recharts** (or plain SVG) | Score charts |
| API | **Node.js + NestJS + TypeScript** | Structured Node.js framework with modules and dependency injection |
| Analyzer service | **Python 3.12 + FastAPI** | Python is the standard for AI/ML work |
| Video processing | **FFmpeg / ffprobe**, OpenCV | Read metadata, extract frames, measure audio |
| Speech-to-text | **faster-whisper** (runs locally, free) | Transcript for captions and hook analysis |
| AI review | **Claude API** (Anthropic Python SDK), vision + structured JSON output | Judges frames and transcript |
| Database | **PostgreSQL** (via Prisma in the API) | Videos, analyses, scores |
| Job queue | **Redis + BullMQ** (Temporal in a later milestone) | Video analysis takes 10–60 s, so it runs in the background |
| File storage | Local folder in development (S3-compatible, e.g. MinIO, later) | Uploaded videos and frames |
| Tests | **Vitest** (scoring library and frontend), **Jest** (NestJS), **pytest** (Python) | |
| Local environment | **Docker Compose** | Runs Postgres, Redis and all services with one command |
| Deployment (stretch) | **Kubernetes** (kind or minikube locally) | Industry standard for running containerised services |

---

## Architecture

```
 ┌──────────────┐  upload / poll   ┌──────────────────┐   job    ┌─────────┐
 │  React app   │ ───────────────► │  NestJS API      │ ───────► │  Redis  │
 │  (web)       │ ◄─────────────── │  (api)           │          │ (queue) │
 └──────────────┘   scores, tips   │  + scoring lib   │          └────┬────┘
                                   └────────┬─────────┘               │ job
                                            │ read/write              ▼
                                            ▼               ┌──────────────────┐
                                    ┌──────────────┐ facts  │ Python analyzer  │
                                    │  PostgreSQL  │ ◄───── │ FFmpeg · Whisper │
                                    └──────────────┘        │ Claude review    │
                                                            └──────────────────┘
```

1. The **web** app uploads the video to the **api**.
2. The api saves the file, creates a `video` row with status `queued`, and puts a job on the Redis queue.
3. The **analyzer** picks up the job and extracts **facts**: metadata, frames, transcript, loudness and the AI review.
4. The api runs the **scoring library** (TypeScript) on those facts: rule checks per platform, then the final scores.
5. The web app polls (later: Server-Sent Events) until the status is `done`, then shows the scores.

> 💡 **Python does the "seeing and hearing", TypeScript does the "judging".** That split keeps
> most of the logic in typed, unit-tested TypeScript.

> 💡 **Mock analyzer first.** Until the Python service exists (milestone 7), the api uses a
> `MockAnalyzer` that returns realistic fake facts. Both implement the same TypeScript interface,
> so swapping the mock for the real service later changes one line. This is a real-world pattern
> called *programming to an interface*.

---

## How a video is scored

### Step 1: Metadata (Python, ffprobe)
Duration, width × height, aspect ratio, frame rate, codec, whether there is an audio track.

### Step 2: Frames (Python, FFmpeg/OpenCV)
- Frames at **0.0 s, 1.0 s, 2.0 s, 3.0 s** (the hook)
- 4–6 more frames spread evenly across the rest of the video
- Scene-change count per 10 seconds (a measure of pacing)
- Average brightness and a blur score per frame

### Step 3: Audio (Python)
- Transcript with timestamps (faster-whisper)
- Loudness (LUFS) and how much of the video is silent
- Words spoken in the first 3 seconds

### Step 4: AI review (Python, Claude)
Send the hook frames, a few later frames, the transcript and the metadata to Claude, and ask for a
**JSON response that matches a fixed schema** (validate it with Pydantic):

```json
{
  "hook":    { "score": 8, "reason": "A face and motion in the first second, with a clear question" },
  "clarity": { "score": 6, "reason": "Product only appears at 7 s" },
  "pacing":  { "score": 7, "reason": "Cuts every 2–3 s" },
  "cta":     { "score": 4, "reason": "No call to action at the end" },
  "frames": [
    { "atSec": 0.0, "note": "Strong opening: face close-up" },
    { "atSec": 3.0, "note": "Text is too small to read on a phone" }
  ],
  "platforms": {
    "tiktok":    { "score": 8, "tips": ["Add on-screen text in the first second"] },
    "instagram": { "score": 7, "tips": ["Add captions"] },
    "youtube":   { "score": 6, "tips": ["Add a clear title card"] },
    "facebook":  { "score": 6, "tips": ["Show the product earlier"] }
  }
}
```

The `frames` notes feed the **frame timeline** in the frontend.

### Step 5: Rule checks (TypeScript, scoring library)
Each platform has a rules file (`packages/scoring/rules/*.json`) so the rules can change without code changes.
Example guidelines to start from:

| Check | TikTok | Instagram Reels | YouTube Shorts | Facebook Reels |
|---|---|---|---|---|
| Aspect ratio | 9:16 | 9:16 | 9:16 | 9:16 |
| Minimum resolution | 1080×1920 | 1080×1920 | 1080×1920 | 1080×1920 |
| Ideal length (guideline) | 15–60 s | 15–90 s | 15–60 s | 15–60 s |
| Captions recommended | Yes | Yes | Yes | Yes |

> ⚠️ Platform limits change often. **Check each platform's current official creator/ads documentation**
> and update the rule files. Do not trust these numbers blindly, and record the date you checked them.

### Step 6: Final score (TypeScript, scoring library)
`final = 0.4 × rule score + 0.6 × AI score` per platform, rounded to 1–10. Keep the weights in config:
milestone 10 tunes them with real data.

---

## Project structure

```
3sReach/
├── README.md
├── docker-compose.yml          # postgres, redis, api, analyzer, web
├── .env.example                # copy to .env, never commit .env
├── packages/
│   └── scoring/                # pure TypeScript: rules, scoring, comparison (+ Vitest tests)
│       ├── src/
│       │   ├── types.ts        # VideoFacts, PlatformRule, PlatformScore, ...
│       │   ├── rules.ts        # check facts against platform rules
│       │   ├── score.ts        # combine rule and AI scores
│       │   └── compare.ts      # A/B comparison
│       └── rules/              # tiktok.json, instagram.json, youtube.json, facebook.json
├── web/                        # React + TypeScript (Vite)
│   └── src/
│       ├── pages/              # Upload, Result, Compare, History
│       ├── components/         # ScoreCard, ScoreChart, FrameTimeline, PlatformBadge, ...
│       ├── state/              # Zustand store (selected frame, compare selection, filters)
│       └── api/                # typed API client + TanStack Query hooks
├── api/                        # NestJS + TypeScript
│   ├── prisma/schema.prisma
│   └── src/
│       ├── videos/             # upload, list, get
│       ├── analyses/           # results
│       ├── analyzer/           # Analyzer interface, MockAnalyzer, HttpAnalyzer
│       └── queue/              # BullMQ producer and worker
├── analyzer/                   # Python + FastAPI
│   ├── app/
│   │   ├── main.py             # FastAPI app
│   │   ├── metadata.py         # ffprobe
│   │   ├── frames.py           # frame extraction, brightness, blur, scene changes
│   │   ├── audio.py            # whisper transcript, loudness
│   │   └── ai_review.py        # Claude call + Pydantic schema
│   └── tests/
└── k8s/                        # Kubernetes manifests (milestone 11)
```

---

## Milestones: build it step by step

Each milestone ends with something that works. **Commit after every milestone.**

### Part A: TypeScript, Node.js and React

| # | Milestone | You learn | Done when |
|---|---|---|---|
| 0 | **Setup**: install Git, Node.js LTS, Docker Desktop; create the repo; enable TypeScript `strict` mode | Tooling | `node -v`, `docker -v` work; `npx tsc --noEmit` passes on an empty project |
| 1 | **Scoring library (pure TypeScript)**: types for video facts and rules, rule checks, weighted scores, A/B comparison. See [core scoring functions](#core-scoring-functions) | Types, interfaces, union types, generics, `map`/`filter`/`reduce`, unit tests with Vitest | `npx vitest` passes with tests for every function |
| 2 | **NestJS API + PostgreSQL with a mock analyzer**: upload endpoint, Prisma schema, `Analyzer` interface with a `MockAnalyzer` that returns fake facts, scoring with the library from milestone 1 | Node.js, NestJS modules/controllers/services, dependency injection, DTO validation, async/await, SQL, ORMs, Jest | Upload with curl or Postman → scores stored in the database and returned by `GET /videos/:id` |
| 3 | **React basics**: upload page with progress, result page with score cards, history page, routing, typed API client, TanStack Query | Components, props, `useState`, `useEffect`, custom hooks, forms, loading and error states, typed API responses | Full flow works in the browser against the mock analyzer |
| 4 | **React advanced**: frame timeline (click a frame to see its note), score charts, compare page (pick two videos), Zustand store for UI state, component tests with Vitest + Testing Library | State management, derived state, memoisation (`useMemo`), performance, accessibility, testing UI | Timeline, charts and compare page work; component tests pass |
| 5 | **Queue + Docker Compose**: move analysis into a BullMQ job and worker (still the mock); one `docker compose up` runs Postgres, Redis, api and web | Background jobs, events, retries, idempotency, containers | Upload returns instantly; the result appears when the job finishes |

### Part B: Python and AI (the real analyzer)

| # | Milestone | You learn | Done when |
|---|---|---|---|
| 6 | **Python: read a video**: install Python 3.12 and FFmpeg; print duration, size and aspect ratio; save the key frames | FFmpeg, Python basics | `python -m app.metadata sample.mp4` prints the data |
| 7 | **FastAPI service + `HttpAnalyzer`**: `POST /analyze` returns real metadata and frames; the api swaps `MockAnalyzer` for `HttpAnalyzer` | REST APIs in Python, connecting services | The browser shows real metadata and real frames in the timeline |
| 8 | **Audio**: transcript with faster-whisper, loudness, silent parts | Speech-to-text | Transcript with timestamps appears on the result page |
| 9 | **AI review**: send frames + transcript to Claude, validate the JSON with Pydantic, retry if invalid | Prompting, vision, structured output, error handling | Real AI scores and frame notes for 3 different test videos |

### Part C: Stretch

| # | Milestone | You learn | Done when |
|---|---|---|---|
| 10 | **Calibration**: let the user enter real views after upload; tune score weights from that data | Product thinking, basic data analysis | Weights are read from data, not hard-coded |
| 11 | **Kubernetes**: run it on kind/minikube; optionally replace BullMQ with Temporal | Kubernetes, workflow engines | `kubectl get pods` shows all services running |

---

## Core scoring functions

Key functions, each covered by unit tests:

| # | Function | What it does |
|---|---|---|
| 1 | `aspectRatioLabel(width, height)` | Returns `"9:16"`, `"1:1"`, `"16:9"` or `"other"` |
| 2 | `checkRules(facts, rules)` | Checks video facts against a platform's rules; returns passed and failed checks |
| 3 | `weightedScore(scores, weights)` | Combines rule and AI scores into a number from 1 to 10 |
| 4 | `topVideos(videos, platform, n)` | Returns the best `n` videos for a platform, without mutating the input |
| 5 | `compareAnalyses(a, b)` | A/B comparison: per-platform winner and the biggest differences |
| 6 | `groupByStatus(videos)` | Groups videos by status: `{ queued: [...], done: [...], ... }` |
| 7 | `silentSegments(loudness, threshold)` | Finds silent parts of the audio as `[{ start, end }]` |
| 8 | `retry(fn, attempts, delayMs)` | Retries a failing async function (used by the queue worker) |
| 9 | `useDebounce(value, ms)` | React hook that debounces the history search box |
| 10 | `<FrameTimeline frames onSelect />` | Keyboard-accessible frame strip on the result page |

---

## Data model

```prisma
model Video {
  id          String     @id @default(uuid())
  filename    String
  storagePath String
  durationSec Float?
  width       Int?
  height      Int?
  status      String     @default("queued") // queued | processing | done | failed
  error       String?
  createdAt   DateTime   @default(now())
  analysis    Analysis?
  results     RealResult[]
}

model Analysis {
  id         String   @id @default(uuid())
  videoId    String   @unique
  video      Video    @relation(fields: [videoId], references: [id])
  facts      Json     // metadata, frames, brightness, pacing, loudness (from the analyzer)
  transcript String?
  aiReview   Json     // validated Claude response (or mock data before milestone 9)
  ruleChecks Json     // per-platform pass/fail list (scoring library)
  scores     Json     // { tiktok: 8, instagram: 7, youtube: 6, facebook: 6 }
  source     String   // "mock" or the Claude model that produced aiReview
  createdAt  DateTime @default(now())
}

// Milestone 10: what actually happened after upload
model RealResult {
  id         String   @id @default(uuid())
  videoId    String
  video      Video    @relation(fields: [videoId], references: [id])
  platform   String
  views      Int
  likes      Int?
  measuredAt DateTime
}
```

---

## API

| Method | Path | Description |
|---|---|---|
| `POST` | `/videos` | Upload a video (multipart). Returns `{ id, status }` |
| `GET` | `/videos` | List videos with status and scores (supports `?search=` and `?status=`) |
| `GET` | `/videos/:id` | One video with its full analysis |
| `GET` | `/videos/:id/frames/:n` | A frame image |
| `POST` | `/compare` | `{ videoIdA, videoIdB }` → side-by-side verdict |
| `POST` | `/videos/:id/results` | Record real views after upload (milestone 10) |

Analyzer (internal only): `POST /analyze` with `{ videoId, path }` → facts.

---

## Configuration

Copy `.env.example` to `.env`:

```bash
# Analyzer: "mock" until milestone 7, then "http"
ANALYZER=mock
ANALYZER_URL=http://localhost:8000

# Claude API (milestone 9): create a key at https://console.anthropic.com (paid, but a test analysis costs only a few cents)
ANTHROPIC_API_KEY=
CLAUDE_MODEL=claude-sonnet-5-5

# Database names can't start with a digit, so the database is called "threesreach"
DATABASE_URL=postgresql://threesreach:threesreach@localhost:5432/threesreach
REDIS_URL=redis://localhost:6379
UPLOAD_DIR=./data/uploads
MAX_UPLOAD_MB=200

# Score weights (milestone 10 tunes these)
WEIGHT_RULES=0.4
WEIGHT_AI=0.6
```

🔒 **Security rules**
- Never commit `.env` or an API key. Put `.env` in `.gitignore` from day one.
- Only the analyzer service needs the API key; the browser never sees it.
- Frames and transcripts are sent to the Claude API. Only analyse videos you own or have permission to use.

---

## Running locally

```bash
# 1. Start Postgres and Redis
docker compose up -d postgres redis

# 2. Scoring library tests
cd packages/scoring
npm install
npx vitest

# 3. API
cd api
npm install
npx prisma migrate dev
npm run start:dev               # http://localhost:3000

# 4. Web
cd web
npm install
npm run dev                     # http://localhost:5173

# 5. Analyzer (from milestone 6)
cd analyzer
python -m venv .venv
.venv\Scripts\activate          # Windows (macOS/Linux: source .venv/bin/activate)
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

After milestone 5: `docker compose up` runs everything.

---

## License

MIT
