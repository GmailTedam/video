# Realtime Video Avatar

A realtime, lifelike video avatar modeled on the user's own likeness, powered by an LLM and built on
**Tavus CVI**. It looks like the user, speaks naturally, and responds with low latency.

## Status (2026-06-24): working end to end

All three phases are built and verified working:

- **Phase 1 (CLI demo).** `setup_demo.py` reads `knowledge/ME.md`, builds a spoken-style system prompt,
  creates a Tavus persona + live conversation, and writes `demo.html` (auto-opens). Text in -> hosted
  LLM with the bio -> realtime talking-head video out.
- **Phase 2 (web app).** `avatar/` (Vite + React + TypeScript) + `avatar/backend/` (zero-dependency Node
  server that holds the API key). Run: `cd avatar && npm install && npm run dev` -> http://localhost:5173.
  Three modes: **Type** (text), **Talk** (mic), **Face to face** (mic + camera = true video-to-video),
  with in-call mic/camera toggles and live device pickers. Backend exposes `POST /api/conversations` and
  `POST /api/conversations/:id/end`; Vite proxies `/api` to it.
- **Phase 3 (custom clone).** The user trained a custom **video** replica of themselves via the Tavus
  portal. It works and looks convincingly like them. Replica id **`r673079e63ce`** ("Ed Jun 23 2026").
  On the **Starter** plan (1 custom replica).

## Requirements (original)

1. Quick to build. 2. As realistic as current tech allows. 3. Video powered by an LLM, lifelike.
4. Looks like the user (personal digital twin). 5. Text input initially.
Nice-to-haves: API/callbacks; low latency; audio input low priority.
Constraints: hosted SaaS; realtime is top priority; bring-your-own-LLM preferred but not at the cost of
latency; likeness from a recorded video; optimize for "wow", not production hardening; budget ~$100.

## Decision: Tavus CVI

Built around cloning a real person (best "looks like me"), cleanest bring-your-own-LLM story with low
turn latency, most API-first, free tier to validate. The demo's "brain" is just `knowledge/ME.md` in the
persona system prompt answered by a Tavus-hosted LLM (`tavus-claude-haiku-4.5`) - lowest latency.
Bring-your-own-LLM is documented in TAVUS.md but not used. HeyGen LiveAvatar was the realism runner-up.

## Architecture / files

- **Root** = `uv` project (Python 3.12): `setup_demo.py`, `knowledge/ME.md`, `pyproject.toml`,
  `.python-version`, `uv.lock`, `.env` (gitignored), `demo.html` (generated, gitignored).
- **`avatar/`** = the web app (own npm project; `npm install && npm run dev` here).
  - `avatar/backend/server.mjs` = zero-dep Node server (reads root `.env` via `../../.env`).
- **`.env` at the project root is SHARED** by Phase 1 (`setup_demo.py`) and Phase 2 (the backend).

## Config (`.env`, gitignored - never commit; it holds the API key)

- `TAVUS_API_KEY` - required.
- `TAVUS_PERSONA_ID` - **auto-managed**: `setup_demo.py` creates a persona from `ME.md` and writes it
  here on first run; `--update-persona` rebuilds and overwrites it. Don't hand-edit.
- `TAVUS_REPLICA_ID` - the face. Currently the user's custom clone `r673079e63ce`. Stock alternatives:
  Anna `r90bbd427f71` (female, the default), Lucas `r5f0577fc829` (male).
- Optional: `TAVUS_LLM_MODEL` (default `tavus-claude-haiku-4.5`), `DEMO_MAX_MINUTES` (10),
  `BACKEND_PORT` (8787).

## Key facts / gotchas learned this session

- **Plans/replicas:** Starter = **1 custom replica**. Stock (system) replicas are unlimited and DON'T
  count against that limit. Switching `TAVUS_REPLICA_ID` to a stock replica is **non-destructive** -
  it never deletes the custom one. Only an explicit delete, or training a 2nd custom replica at the cap,
  removes it. Upgrading raises the custom-replica limit.
- **Concurrency:** low tiers allow ~1 concurrent conversation. Both `setup_demo.py` and the backend
  **auto-end any active conversation before creating a new one** (else: "maximum concurrent
  conversations"). A conversation is a single live session; leaving/timeout destroys the Daily room, so
  you re-run to start a new one. `participant_left_timeout` = 120s (grace for a quick refresh).
- **Persona:** bakes `ME.md` (system prompt) at creation. Edit `ME.md` -> run `--update-persona` to apply.
  No explicit TTS layer (default voice); rebuild the persona around a replica to keep face + voice consistent.
- **LLM / latency tuning (settled).** Model is `tavus-claude-haiku-4.5` - best tone/character fit for this
  persona. Tried `tavus-gemini-3-flash` (good, a touch snappier, but weaker personality) and reverted;
  `tavus-gpt-oss` is fastest-rated but is a *reasoning* model (low/med/high effort, no zero) - odd fit for
  chat-only. In persona `layers.conversational_flow`, **`turn_taking_patience: "low"` is KEPT** - shortens
  the wait after the user stops before the avatar replies (tradeoff: too low can cut you off mid-pause).
  **Do NOT set `speculative_inference: true`** - it caused a reproducible ~2s audio/lip-sync drift in
  Tavus' render (left off). Tavus publishes no per-model latency numbers, only relative speed icons.
- **Short-answer prompt:** the system prompt is tuned for one-to-two-sentence replies, with a re-anchor
  line AFTER the baked `{knowledge}` bio (it's the last thing the model reads, so recency was pulling it
  toward reciting background). Trimming `ME.md` aids small-model coherence, not latency (prompt is ~1k
  tokens, well under Tavus' ~5k sweet spot).
- **Frontend/Daily (avatar/src/App.tsx):** create the conversation BEFORE the Daily call object (so a
  failed fetch leaves nothing to tear down), and sequence `leave()` -> `destroy()`. No `StrictMode`
  (avoids a duplicate Daily instance). "Face to face" joins with mic+camera; "Type" joins with neither
  (no permission prompt).
- **A "fetch failed" / network error seen once was a transient connectivity blip - NOT a code bug and
  NOT Tavus down.** Verified the machine reaches `tavusapi.com` reliably (AWS us-west, `54.193.x.x`).
  Note: the command sandbox blocks outbound network, so diagnose connectivity with the sandbox disabled
  or have the user run it.

## Design rules (UI)

Palette: amber `#ecad0a`, blue `#209dd7`, purple `#753991`, over grays. Sharp, modern, clean, **flat**.
- **No gradients and no left-border accent lines** - the user considers these classic "LLM tells".
- **Exception:** the small conic-gradient logo brand mark is kept deliberately (the user likes it).
- Top accent = a single solid **amber** line. Mode buttons are solid brand colors (Type=amber,
  Talk=blue, Face to face=purple). Hero accent word is solid blue.

## Key references

- **[TAVUS.md](./TAVUS.md)** - full Tavus reference (architecture, replica training, persona/LLM/voice,
  runtime conversation control, API, CLI, webhooks, deep-links to docs). The user has edited it too.
- **[README.md](./README.md)** - the shareable 3-phase build guide (Phase 1 detailed; Phase 3 records
  the portal-based replica training and points at the Phase 2 site to run it).
- **[setup_demo.py](./setup_demo.py)** - Phase 1 script.

## Working on this

- Python: `uv run setup_demo.py` (uv project, Python 3.12); `uv run python ...` for one-offs.
- Web app: `cd avatar && npm run dev` (starts backend + Vite together); `npm run build` to type-check.
- Methodical debugging: prove the root cause before fixing (per global CLAUDE.md). The network
  misdiagnosis above is a reminder to verify, not guess.
