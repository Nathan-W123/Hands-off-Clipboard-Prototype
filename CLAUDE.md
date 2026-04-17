# Y Combinator Cactus Hackathon — Project Context

## Project Overview

Offline-first, voice-driven data capture tool using the Cactus SDK. Three core pillars:

1. **Interactive Checklists** — voice-driven step confirmation with local state machine
2. **Structured Database Entries** — spoken field notes parsed into typed JSON schemas

Core value proposition: convert spoken field notes into structured JSON payloads in extreme, offline, or noisy environments (aviation, agriculture, manufacturing) with zero cloud dependency at the point of capture.

---

## Tech Stack

| Layer | Technology | Notes |
| --- | --- | --- |
| Frontend | React Native (Expo) + TypeScript | Strict typing mandatory, no `any` |
| On-device AI | Cactus SDK (React Native bindings) v1.12 | STT + LLM inference, fully offline — pin version, do not upgrade during sprint |
| Local database | SQLite via `expo-sqlite` | Primary offline store and sync queue |
| Cloud database | Supabase (PostgreSQL) | Receives data when connectivity returns |
| Backend | Python | Cloud-side processing and export integrations only |
| UI | React Native core components | No heavy UI libraries |

---

## Architectural Constraints

**1. Offline-first, always** — No feature may require a network connection. App must be fully functional in airplane mode. Data written to SQLite first, always.

**2. Offline sync queue (critical)** — Supabase has no built-in offline SDK. Build a `sync_queue` table in SQLite. Flow: `Voice → Cactus STT + LLM → JSON → SQLite → [signal detected] → Supabase`. Use `@react-native-community/netinfo` to detect connectivity. Drain queue with exponential backoff on reconnect. Mark records `synced: true` on success. Never delete local records — SQLite is the source of truth.

**3. Zero latency UI** — No loading spinners. Instant visual confirmation on parse. Optimistic UI throughout. Sync status is a background indicator only.

**4. Noise resilience** — Push-to-activate toggle is the primary input mode — press once to start listening, press again to stop and parse. Pre-process audio locally before passing to Cactus STT. Test against hangar, wind, and machinery audio samples.

**5. Strict TypeScript** — All schemas defined as interfaces in `/src/types/schemas.ts`. No `any`. All Cactus SDK responses typed before use. Shared interfaces used for both SQLite writes and Supabase payloads.

---

## JSON Schema Conventions

Every voice-parsed record must include: `id`, `schema_version`, `captured_at` (ISO 8601), `fields` (typed key-value map), `raw_transcript`, `confidence_score` (0–1), `validated`, `synced`. Optional: `location` (lat/lng). All defined as strict TypeScript interfaces in `/src/types/schemas.ts`. Never redefine inline.

---

## Smart Validation Rules

On-device LLM must flag: confidence below 0.7 (re-prompt), out-of-range values (check schema min/max), missing required fields (block commit), ambiguous values (confirm before writing), type mismatches (reject and re-prompt). Feedback must be spoken via TTS immediately, shown in parsed JSON preview, and non-blocking — user can override and commit.

---

## Voice Parser Prompt Convention

Write all few-shot examples before any parser code — this is the ground truth. The Cactus LLM system prompt must include: active schema fields with types and valid ranges, minimum 10 few-shot examples (raw transcript → JSON), explicit instruction to return `confidence_score` (0–1) and `raw_transcript` on every response, and instruction to flag uncertain values rather than guess. Format: `Input: "<transcript>" → Output: { ...structured JSON }`.

---

## Demo Templates (Pre-loaded)

| Template | Domain | Key fields |
| --- | --- | --- |
| RV-14 Pre-flight Checklist | Aviation | Step label, confirmed, notes |
| Crop Field Scout | Agriculture | Species, count, GPS, severity rating |
| Manufacturing Defect Report | Industrial | Part ID, defect type, dimensions, location |

Each defined as a TypeScript interface and a corresponding Supabase table.

---

## UI/UX Constraints

- **Aesthetic**: Apple-inspired neo-minimalism. Massive whitespace. Clean typography.
- **Primary action**: Push-to-activate toggle — press once to begin listening, press again to stop and parse
- **Colour**: Dark mode supported. Monochrome base, one accent colour only
- **Animations**: State transitions only — no decorative animation
- **Typography**: System font (SF Pro / Roboto). Two weights only
- **Haptics**: `expo-haptics` on parse success, validation failure, sync completion

---

## Project Structure

`/src
  /types        schemas.ts
  /services     cactus.ts · sqlite.ts · syncQueue.ts · supabase.ts · validation.ts
  /components   VoiceCapture.tsx · ParsedPreview.tsx · ChecklistRunner.tsx · SyncStatus.tsx
  /screens      Home.tsx · Checklist.tsx · History.tsx
  /templates    preflight.json · cropscout.json · defectreport.json`

---

## AI Assistant Rules

1. **Strict TypeScript frontend** — no `any`, no implicit types
2. **SQLite first** — never write directly to Supabase, always queue locally
3. **Never assume connectivity** — all Supabase calls must handle the offline case
4. **Use shared interfaces** from `/src/types/schemas.ts` — never redefine inline
5. **Python backend only** — for Supabase edge functions and export integrations
6. **Components under 150 lines** — extract logic to services
7. **No loading spinners** — optimistic UI patterns only
8. **Validate before committing** — run validation before every SQLite write

---

## Build Priority (21-hour sprint)

1. Cactus STT → LLM → typed JSON — **hrs 0–8**
2. SQLite schema + write service — **hrs 2–8** (parallel)
3. Wire parser output into SQLite — **hrs 8–10**
4. Validation layer + TTS feedback — **hrs 10–14**
5. UI shell + VoiceCapture component — **hrs 10–14** (parallel)
6. Supabase sync queue — **hrs 14–17**
7. Demo templates tested end-to-end — **hrs 17–19**
8. Full offline → sync demo rehearsal — **hrs 19–21**
9. Python export integrations — **post-hackathon**

---

## Known Risks

| Risk | Mitigation |
| --- | --- |
| Cactus model cold load (10–30s) | Loading screen on first launch only, cache model after |
| LLM parser accuracy | Write 15+ few-shot examples before any code |
| Supabase sync queue complexity | Time-box 3hrs max — manual sync button acceptable for demo |
| TTS latency | Test on real hardware hour 1, not hour 10 |
| Noise kills STT | Push-to-activate toggle sidesteps this for the demo |
