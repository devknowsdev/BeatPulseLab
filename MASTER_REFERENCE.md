# BeatPulse / TuneTag Annotator — Master Reference Document
*Consolidated from all handover docs · Last session: 21 February 2026*

> ⚠️ **Naming note:** The project folder is `tunetag-annotator` and the deep research doc refers to "BeatPulse Annotator", but all handover and UI docs use "TuneTag Annotator". Clarify the canonical name before any public-facing work.

---

## 1. Project Overview

| Field | Detail |
|---|---|
| **App name** | TuneTag Annotator (folder: `tunetag-annotator`) |
| **Purpose** | Real-time audio annotation for music supervisors and sound editors. Annotate Spotify tracks with timestamped notes, record voice memos, export to Excel. |
| **Primary user** | Music industry professional — ADHD/Autistic — needs low-friction, high-predictability, calm UI |
| **Stack** | React 18 + TypeScript + Vite 5 |
| **Live URL** | https://tunetag.devknowsdev.workers.dev/ |
| **Hosting** | Cloudflare Pages via Wrangler |
| **Local dev** | `npm run dev` → http://localhost:5173 (**always use `localhost`, NOT `127.0.0.1`** — Web Speech API requires it) |
| **Repo** | GitHub (`main` branch, auto-deploys on push) |

---

## 2. Local Paths

```
/Users/duif/DK APP DEV/tunetag-annotator     ← live project
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator BACKUP.zip ← original backup
```

---

## 3. Source Structure

```
src/
├── App.tsx                    ← root component, all state, phase routing
├── main.tsx                   ← entry point
├── index.css                  ← all CSS vars + component classes (~770 lines)
├── global.d.ts                ← global type declarations (incl. Spotify.Player types)
├── types/index.ts             ← shared TypeScript interfaces (incl. RecordingEntry)
│
├── components/
│   ├── SetupScreen.tsx        ← ✅ NEW — replaces ApiKeyGate (completed, committed)
│   ├── ApiKeyGate.tsx         ← 🗑 OLD — safe to delete
│   ├── PhaseListening.tsx     ← largest file (1254 lines) — NEEDS REFACTOR
│   ├── PhaseMarkEntry.tsx
│   ├── PhaseGlobal.tsx
│   ├── PhaseReady.tsx
│   ├── PhaseReview.tsx
│   ├── PhaseSelect.tsx
│   ├── SpotifyPlayer.tsx
│   ├── RecordingsPanel.tsx
│   ├── LintPanel.tsx
│   ├── HowToUse.tsx
│   ├── AppSidebar.tsx
│   └── WaveformScrubber.tsx
│
├── hooks/
│   ├── useAnnotationState.ts  ← all app state: annotations, phase, autosave
│   ├── useAudioRecorder.ts
│   ├── useAudioDevices.ts
│   ├── useDictation.ts
│   ├── useMicMeter.ts
│   ├── useSpotifyPlayer.ts
│   ├── useKeyboardShortcuts.ts
│   ├── useTimer.ts            ← single timer, lives in App.tsx only
│   └── index.ts               ← barrel file
│
└── lib/
    ├── schema.ts              ← track definitions, categories, section types, tags
    ├── spotifyApi.ts          ← Spotify Web API wrappers
    ├── spotifyAuth.ts         ← PKCE auth — REDIRECT_URI still needs updating
    ├── excelExport.ts         ← ExcelJS write logic
    ├── lintAnnotation.ts      ← pre-export validation
    ├── polishText.ts          ← OpenAI API for transcript polish
    ├── tagPacks.ts
    ├── tagLibrary.ts
    ├── tagImport.ts
    ├── phraseBuilder.ts
    └── loadResearchedPacks.ts
```

---

## 4. Architecture

### 4.1 State Model

```typescript
// AppState — single source of truth, persisted to localStorage
interface AppState {
  annotations: Record<number, TrackAnnotation>; // keyed by track id
  activeTrackId: number | null;
  phase: Phase;
  markEntryDraft: MarkEntryDraft | null;
  globalCategoryIndex: number;
  globalOnSummary: boolean;
  timerRunning: boolean;
}

// TrackAnnotation — per-track (nested in AppState.annotations)
interface TrackAnnotation {
  track: Track;
  annotator: string;
  timeline: TimelineEntry[];       // up to 10 entries
  global: Partial<GlobalAnalysis>; // 9 categories
  status: 'not_started' | 'in_progress' | 'complete' | 'skipped';
  elapsedSeconds: number;
  resumePhase?: Phase;
  startedAt?: number;
  completedAt?: number;
  lastSavedAt?: number;
  skipReason?: string;
}
```

### 4.2 Phase Machine

```
select → ready → listening ⇄ mark_entry
                     │
                     ↓
                  global → review → select
```

- Only one phase renders at a time — **except** `listening` + `mark_entry` co-render (mark_entry is a fixed overlay; listening stays mounted to preserve scroll)
- `isActive={phase === 'listening'}` gates keyboard shortcuts in PhaseListening while mark_entry is open
- Phase is persisted in AppState; `resumePhase` is persisted per-track

### 4.3 Timer

- `useTimer` lives **only** in App.tsx
- Props thread down: `elapsedSeconds`, `isTimerRunning`, `timerStart`, `timerPause`
- Timer ticks → `updateElapsedSeconds(id, secs)` → persisted to TrackAnnotation
- A `useEffect` keyed on `[activeTrackId, phase, timerRunning]` rehydrates elapsed on track/phase change

### 4.4 Autosave

- Every `setAppState` call debounces a 500ms localStorage write
- Immediate flush on: `pagehide`, `beforeunload`, hook unmount
- On app load: if saved state found, resume banner shown
- `resumeSavedState()` is **synchronous** so timer restores before React re-render

### 4.5 Key Design Decisions

| Decision | Reason |
|---|---|
| No router | Phase is pure state; URL routing adds complexity without benefit |
| No Redux/Zustand | useAnnotationState is the complete state layer |
| No backend | All data is local; ExcelJS runs in-browser |
| Timer lives in App | Prevents duplicate timers and stale-closure bugs |
| Co-render listening + mark_entry | Preserves scroll position in listening |
| resumePhase per track | Global phase is always 'select' when PhaseSelect renders |
| Synchronous resumeSavedState | Allows timer restore before re-render |

---

## 5. Pending Changes (Do These First)

| # | File | Action |
|---|------|--------|
| 1 | `src/lib/spotifyAuth.ts` | Replace hardcoded REDIRECT_URI with `window.location.origin + '/callback'` |
| 2 | `src/components/ApiKeyGate.tsx` | Delete (replaced by SetupScreen) |
| 3 | `netlify.toml` | Delete (no longer on Netlify) |
| 4 | `src/.DS_Store` | Delete + add `**/.DS_Store` to `.gitignore` |

**spotifyAuth.ts fix:**
```ts
const REDIRECT_URI = window.location.origin + '/callback'
```

---

## 6. Planned Refactor — PhaseListening.tsx

File is at 1254 lines. Run each prompt one at a time; run `npx tsc --noEmit` after each step.

| Step | Action |
|------|--------|
| 1 | Housekeeping (gitignore, delete netlify.toml, delete ApiKeyGate.tsx) |
| 2 | Extract `src/hooks/useMicMeter.ts` |
| 3 | Extract `src/hooks/useAudioRecorder.ts` |
| 4 | Extract `src/hooks/useDictation.ts` |
| 5 | Extract `src/components/RecordingsPanel.tsx` |
| 6 | Final cleanup — PhaseListening < 300 lines, barrel file `src/hooks/index.ts` |
| 7 | Commit and push |

---

## 7. Transport Synchronisation (Priority P0 — Core Bug)

The current implementation has a timer racing Spotify rather than being driven by it. These are the highest-priority fixes.

### 7.1 Problems to Fix

- Timer and Spotify run independently — drift accumulates
- `playTrack()` doesn't send `position_ms` — always starts from 0
- Track duration is hard-coded at 300s
- Waveform depends on a potentially 403'd Audio Analysis endpoint

### 7.2 Unified Transport Architecture

All transport actions go to Spotify Web API first; the UI timer is continuously synced from Spotify's playback state. Drift > 0.5–1s triggers a correction.

```
User action (play/pause/seek)
  → TransportController
    → activateElement() [for autoplay restrictions]
    → Spotify Web API (authoritative)
    → UI timer (follows Spotify)
    → Poll every ~1s: if |timer.sec - state.progress_ms/1000| > 1s → correct timer
```

### 7.3 spotifyApi.ts Changes

```ts
export async function playTrack(
  spotifyId: string, deviceId: string, token: string, opts?: { positionMs?: number }
): Promise<void> {
  const body: any = { uris: [`spotify:track:${spotifyId}`] };
  if (opts?.positionMs !== undefined) body.position_ms = Math.max(0, Math.floor(opts.positionMs));
  await fetch(
    `https://api.spotify.com/v1/me/player/play?device_id=${encodeURIComponent(deviceId)}`,
    { method: 'PUT', headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' }, body: JSON.stringify(body) }
  );
}

export async function pausePlayback(deviceId: string, token: string) {
  await fetch(`https://api.spotify.com/v1/me/player/pause?device_id=${encodeURIComponent(deviceId)}`,
    { method: 'PUT', headers: { Authorization: `Bearer ${token}` } });
}

export async function seekPlayback(deviceId: string, token: string, positionMs: number) {
  await fetch(
    `https://api.spotify.com/v1/me/player/seek?position_ms=${Math.max(0, Math.floor(positionMs))}&device_id=${encodeURIComponent(deviceId)}`,
    { method: 'PUT', headers: { Authorization: `Bearer ${token}` } });
}
```

### 7.4 App.tsx Transport Handler Changes

```tsx
const tokenRef = useRef(token);
useEffect(() => { tokenRef.current = token; }, [token]);

async function handlePlay() {
  setTimerRunning(true);
  if (deviceId && spotifyToken) {
    await transferPlayback(deviceId, spotifyToken);
    await playTrack(activeSpotifyId, deviceId, spotifyToken, { positionMs: timer.elapsedSeconds * 1000 });
  }
  timer.start();
}
function handlePause() {
  timer.pause(); setTimerRunning(false);
  spotifyPlayer.pause().catch(() => {});
}
function handleStop() {
  timer.pause(); timer.setSeconds(0); setTimerRunning(false);
  spotifyPlayer.pause().catch(() => {});
  spotifyPlayer.seek(0).catch(() => {});
}

// Drift correction
useEffect(() => {
  if (!timerRunning || !spotifyPlayer.isPlaying) return;
  const spotifySec = Math.floor(spotifyPlayer.position / 1000);
  const drift = Math.abs(spotifySec - timer.elapsedSeconds);
  if (drift >= 1) timer.setSeconds(spotifySec);
}, [spotifyPlayer.position, spotifyPlayer.isPlaying, timer.elapsedSeconds, timerRunning]);
```

### 7.5 global.d.ts Addition

```ts
interface SpotifyPlayer {
  activateElement(): Promise<void>;
  // ... existing methods
}
```

### 7.6 Track Duration Fix

Remove hard-coded `durationSeconds = 300`. Use:
```tsx
const fullDuration = analysis?.track?.duration ?? Math.floor(spotifyPlayer.duration / 1000) || 0;
```

### 7.7 Refresh Token

Implement PKCE refresh before 1-hour token expiry. Use a `tokenRef` in `useSpotifyPlayer` so the SDK always uses the latest token.

---

## 8. Waveform Strategy

Try in order; fall back gracefully:

| Approach | Method | Risk |
|---|---|---|
| **A — Audio Analysis** | `GET /audio-analysis/{id}` → segment loudness | Often returns 403 for dev apps (deprecated endpoint) |
| **B — User Audio Upload** | Web Audio API decodes user's file | Requires user to have the file; must not distribute audio |
| **C — Pseudo-waveform** | Deterministic pattern from track ID | Not accurate; purely illustrative — safest fallback |

Always label the waveform type clearly in UI. Always enable scrubbing regardless of waveform source.

**Policy note:** Spotify's "no synchronisation with visual media" rule is a grey area for waveform display. BeatPulse's annotation-only purpose (not video output) reduces risk, but review before any public launch.

---

## 9. Required Spotify Scopes

| Scope | Used For |
|---|---|
| `user-modify-playback-state` | `/play`, `/pause`, `/seek` |
| `user-read-playback-state` | Reading `progress_ms`, identifying current track |

---

## 10. Spotify Developer Setup

**Dashboard:** https://developer.spotify.com/dashboard  
**App name:** TuneTag (existing — do NOT delete; 24hr creation limit in effect)

**APIs to enable:**
- ✅ Web API
- ✅ Web Playback SDK
- ❌ Android/iOS/App Remote SDK — keep off

**Redirect URIs (add both):**
```
https://tunetag.devknowsdev.workers.dev/callback
http://localhost:5173/callback
```

**Important:** App is in Development Mode. Add Spotify account email as a test user (Users and Access section) before testing live deployment. For public access, submit a quota extension request.

---

## 11. Cloudflare Deployment

**Live URL:** https://tunetag.devknowsdev.workers.dev/  
**Config:** `wrangler.jsonc` in project root  
**Deploy:** automatic on push to `main`

```json
{
  "name": "tunetag-annotator",
  "compatibility_date": "2026-02-21",
  "assets": {
    "directory": "./dist"
  }
}
```

For SPA routing (handles `/callback`), add to `wrangler.jsonc`:
```json
"assets": {
  "directory": "./dist",
  "not_found_handling": "single-page-application"
}
```

---

## 12. CI/CD Recommendations

### GitHub Actions (CI on every PR/push)
```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with: {node-version: '18', cache: 'npm'}
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run build
```

- Protect `main` branch — require CI pass before merge
- Store `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` as GitHub secrets
- Use `cloudflare/wrangler-action` for auto-deploy on merge to `main`
- Tag stable commits; rollback by reverting to tag and re-deploying

---

## 13. API Validation (cURL Checks)

```bash
export TOKEN="<SPOTIFY_OAUTH_TOKEN>"
export DEVICE_ID="<DEVICE_ID>"

# List available devices
curl -s -H "Authorization: Bearer $TOKEN" https://api.spotify.com/v1/me/player/devices | jq .

# Start playback at 10s
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  "https://api.spotify.com/v1/me/player/play?device_id=$DEVICE_ID" \
  -d '{"uris":["spotify:track:4iV5W9uYEdYUVa79Axb7Rh"],"position_ms":10000}'

# Get current playback state
curl -s -H "Authorization: Bearer $TOKEN" https://api.spotify.com/v1/me/player | jq '.progress_ms, .is_playing'

# Pause
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  "https://api.spotify.com/v1/me/player/pause?device_id=$DEVICE_ID"

# Seek to 30s
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  "https://api.spotify.com/v1/me/player/seek?device_id=$DEVICE_ID&position_ms=30000"
```

---

## 14. Manual Test Plan

1. **Login & setup:** `npm run dev` → http://localhost:5173 → log in via PKCE → confirm redirect URI matches
2. **Basic playback:** Select track → Start Listening → verify Spotify plays from 0:00, timer starts
3. **Play/Pause sync:** Press Space → confirm Spotify and timer pause/resume together
4. **Seeking:** Click waveform / use ±10s buttons → verify both elapsed time and Spotify position update
5. **Stop:** Click Stop → Spotify pauses, progress resets to 0:00, timer resets
6. **Waveform:** Test various track lengths — confirm full waveform or graceful fallback
7. **Token expiry:** After ~1h, confirm refresh works or app prompts re-auth
8. **Error cases:** Test no-network and non-Premium — verify clear error messages

---

## 15. Debug Checklist

- `spotifyPlayer.isReady` → verify it becomes `true` on SDK `ready` event
- `device_id` → confirm correct ID via `getAvailableDevices()`; verify `transferPlayback` didn't 403
- Browser console: call `spotifyPlayer.getCurrentState()` → check `position` and `duration`
- Compare `progress_ms` from Web API with SDK position — they should match within ~1s
- Drift correction `useEffect` → add `console.log` to confirm it fires each second
- Network inspector → inspect Web API calls; compare with cURL tests

---

## 16. Planned: Flow Mode (Not Yet Built)

A stripped-back, continuous annotation mode. No modals — designed for uninterrupted listening.

**UI elements confirmed:**
- Track progress bar (scrubbable)
- ⏮ -10s / ⏭ +10s skip buttons
- Play / Pause
- Live audio level meter (mic monitoring)
- Tag buttons — silent timestamp stamp, no modal
- **Smart Dictate toggle** — continuous, auto-stamps transcript + timestamp as timeline notes
- **Speech-to-Text toggle** — live transcript visible on screen
- Read-only live-updating timeline

**What's NOT in Flow Mode:** No mark-moment modal, no global ratings panel, no recordings panel

**Architecture decision needed:** New `flow` phase in App.tsx router (recommended) vs toggle within existing `listening` phase

**Smart Dictate logic (first pass):**
- Continuous `SpeechRecognition` (not one-shot)
- On each result: capture transcript + current timestamp → save as timeline entry (category: "Note")
- No confirmation step — saves and continues

---

## 17. Design System

### Colour Tokens (from index.css)

| Token | Hex / Value | Usage |
|---|---|---|
| `--bg` | `#0a0a0a` | Page background |
| `--surface` | `#111111` | Cards, panels |
| `--surface-raised` | `#171717` | Elevated elements, modals |
| `--border` | `#1e1e1e` | Dividers |
| `--border-active` | `#2e2e2e` | Focused borders |
| `--amber` | `#f59e0b` | Primary accent — CTAs, timer, key labels |
| `--amber-bg` | `#1c1500` | Amber-tinted backgrounds |
| `--amber-glow` | `rgba(245,158,11,0.1)` | Subtle glow |
| `--text` | `#d4cbbe` | Body text (warm off-white) |
| `--text-muted` | `#7a7268` | Secondary text |
| `--text-dim` | `#3e3c38` | Tertiary, disabled |
| `--error` | `#ef4444` | Errors, destructive |
| `--error-bg` | `#1c0a0a` | Error-tinted backgrounds |
| `--success` | `#22c55e` | Success states |

### Typography

| Token | Stack | Usage |
|---|---|---|
| `--font-mono` | JetBrains Mono | Labels, timestamps, counters |
| `--font-serif` | Georgia / Times New Roman | User-written content, annotations |
| `--font-display` | Playfair Display | Track names, screen titles |

### Button Classes
`btn-primary`, `btn-ghost`, `btn-small`, `btn-destructive`, `btn-link`, `label`

### Layout
- Max content width: **580px**, centred
- Border radius: `--radius: 8px` (cards), `--radius-pill: 100px` (chips)
- Transitions: 150–200ms fade/opacity only — no scale transforms

---

## 18. Session Storage Keys

| Key | Purpose |
|---|---|
| `spotify_api_key` | Spotify client token |
| `openai_api_key` | OpenAI key (Whisper + text polish) |
| `tunetag_api_key_gate_done` | Whether setup screen completed |

---

## 19. Useful Terminal Commands

```bash
# Navigate to project
cd "/Users/duif/DK APP DEV/tunetag-annotator"

# Run locally (always localhost, not 127.0.0.1)
npm run dev

# TypeScript check
npx tsc --noEmit

# Build for production
npm run build

# Commit and push
git add .
git commit -m "your message"
git push

# Recent commits
git log --oneline -5

# File line counts (spot large files)
wc -l src/**/*.{ts,tsx,css} src/*.{ts,tsx,css} | sort -rn

# Folder sizes
du -sh src/* | sort -rh

# Create handover zip (excluding node_modules/dist)
zip -r beatpulse_handover.zip . -x "node_modules/*" "dist/*" "*.log"
```

---

## 20. Risk Register

| Risk | Level | Mitigation |
|---|---|---|
| API race conditions (async Spotify calls) | High | Serialize with `await`; confirm state via `getCurrentState()` before trusting |
| Token expiry after 1 hour | High | Implement PKCE refresh; use `tokenRef` in SDK |
| Audio Analysis endpoint returning 403 | Medium | Waveform fallback strategy (upload / pseudo) |
| Spotify "sync with visual media" policy | Medium | Emphasise annotation-only use; keep waveform optional |
| Non-Premium users get `account_error` | Medium | Detect SDK error; show clear "Premium required" message |
| Timer drift under network load | Low | Correction threshold 0.5–1s; consider manual sync button |
| Bundle size (currently 1.17MB) | Low | Will improve naturally after refactor enables code-splitting |

---

## 21. Recommended Next Steps (Priority Order)

### Immediate (this session)
1. **Fix `spotifyAuth.ts` REDIRECT_URI** — dynamic `window.location.origin + '/callback'`
2. **Delete `ApiKeyGate.tsx`** and **`netlify.toml`**
3. **Add `**/.DS_Store` to `.gitignore`** and delete existing .DS_Store files
4. **Begin `PhaseListening.tsx` refactor** — follow the 7-step plan in §6

### Short term (next 1–2 sessions)
5. **Transport overhaul** — implement unified TransportController, add `position_ms` to `playTrack()`, add `pausePlayback()` / `seekPlayback()` wrappers, wire drift correction (§7)
6. **Fix hard-coded `durationSeconds = 300`** — use `spotifyPlayer.duration` or track metadata
7. **Set up Vitest** — unit tests for transport logic and drift correction
8. **Implement PKCE token refresh** — prevent 1-hour session failures

### Medium term
9. **CI/CD** — GitHub Actions + Wrangler deploy action (§12)
10. **Waveform strategy** — try Audio Analysis, fallback to upload, then pseudo-waveform (§8)
11. **Flow Mode** — new phase in App.tsx router (§16)
12. **UX polish** — replace `window.confirm`/`window.prompt` with custom modal components

### Before any public launch
13. **Spotify quota extension request** — explain app purpose for non-dev access
14. **Review Spotify developer policy** re: waveform sync feature
15. **Add test users** to Spotify developer app (Users and Access)
