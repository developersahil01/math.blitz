# RRB JE Math Blitz

A fast, gamified Mathematics practice app for RRB JE with a **deterministic, template-based question generation engine**. Every question is generated on the fly from a generator (template) + seed — no AI, no pre-stored question bank, no answer leakage.

- **Infinite practice**: 39 generators across all 24 required topics
- **Deterministic**: same `templateId + difficulty + seed` always reproduces the exact same question
- **No answer leakage**: correct answer/explanation are only ever sent to the client *after* answering
- **Adaptive difficulty**: 3 correct in a row → move up a level; 2 wrong → move down
- **Gamified**: XP (base + difficulty + speed + streak bonuses), levels, day streaks, achievements, leaderboard
- **Modes**: Quick Blitz, Topic Drills, Sprint (timed), Weak-Area (auto-targets worst topic), Daily Challenge (same set for everyone each day), Exam (timed, difficulty mix)
- **Works without a database**: in-memory store is the default; MongoDB is optional

## Project layout

```
question-engine/   Pure CommonJS Node module (no deps) — the question engine
server/            Express API + services + stores (in-memory / MongoDB)
client/            React (Vite) mobile-first SPA
```

## Quick start

```bash
npm run install:all     # install root + server + client deps
npm run dev:server      # API on http://localhost:4000 (in-memory store)
npm run dev:client      # Vite dev server on http://localhost:5173 (proxies /api)
```

Production-style single deploy (server serves the built client):

```bash
npm run build           # build client into client/dist (server serves it)
npm start               # serve API + client on http://localhost:4000
npm run seed:demo       # simulate a few demo players (in-memory)
```

## Tests

```bash
npm test                # engine + server suites
npm run test:engine     # question generator invariants + determinism
npm run test:server     # HTTP API integration tests
```

Each generator is stress-tested (10,000+ generated questions, `STRESS_COUNT` override), validating invariants, unique options, and a single correct answer.

## How the engine works

- `question-engine/core/seed.js` — `randomSeedString()` + `xmur3`/`mulberry32` RNG
- Every generator is `{ id, topic, subtopic, difficulties, generate({difficulty, rng}), verify? }`
- `generateQuestion` runs a generator, retries on duplicate signatures / invalid options, and produces a self-contained `question` object:
  `{ text, options, correctIndex, correctAnswer, explanation, parameters, estimatedTime, seed, signature, templateId, topic, difficulty }`
- `signature` (templateId + canonical params) prevents repeated questions within a session
- Distractors (`core/distractors.js`) are drawn from the generator's own answer pool and `core/options.js` dedupes + shuffles so there is exactly one correct option

## API (main routes)

| Route | Purpose |
| --- | --- |
| `POST /api/auth/device` | create/login a device user (`deviceId`, optional `name`) |
| `GET /api/topics`, `GET /api/generators` | registry |
| `POST /api/practice/start` | start `quick\|topic\|sprint\|weak-area\|exam\|daily` session |
| `GET /api/practice/:id` | resume — current question (answers hidden) |
| `POST /api/practice/:id/answer` | submit `questionIndex, choice [, timeTakenMs]`; `choice` may be `null` for time-out |
| `GET /api/practice/:id/result` | full review after completion |
| `GET /api/daily-challenge`, `POST /api/daily-challenge/start` | daily challenge |
| `GET /api/user/stats`, `/user/topic-stats`, `/user/achievements` | personal stats |
| `GET /api/leaderboard`, `/api/exam/presets` | social / exam config |
| `/api/admin/*` | dev tools: list/test/sample generators, toggle availability, global stress run |

All user-scoped routes take the user id via the `X-User-Id` header (the client stores a deviceId in localStorage).

## Configuration

- `PORT` (default `4000`)
- `MONGO_URI` — if set, the server uses MongoDB via Mongoose; otherwise the in-memory store is used (seeded on demand, data lost on restart)