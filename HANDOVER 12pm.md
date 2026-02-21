# BeatPulse Annotator — Project Handover Doc
*Last updated: 21 February 2026 — Session 2*

---

## Project Overview

**App:** BeatPulse Annotator
**Purpose:** Real-time audio annotation tool for music supervisors and sound editors. Annotate Spotify tracks with timestamped notes, record voice memos, export to Excel.
**Stack:** React + TypeScript + Vite
**Repo:** GitHub (main branch, always up to date)
**Live URL:** https://beatpulselab.devknowsdev.workers.dev/
**Hosting:** Cloudflare Pages (via Wrangler)
**Local dev:** `npm run dev` → http://localhost:5173 (use localhost NOT 127.0.0.1 — Web Speech API requires it)

---

## Key File Paths

```
/Users/duif/DK APP DEV/BeatPulseLab/beatpulse-annotator          ← live project
/Users/duif/DK APP DEV/BeatPulseLab/beatpulse-annotator BACKUP.zip ← original backup
```

### Source structure
```
src/
├── App.tsx                          ← root, all state, phase routing
├── main.tsx                         ← entry point
├── index.css                        ← all CSS vars, dark/light theme (770 lines)
├── global.d.ts                      ← global type declarations
├── types/index.ts                   ← shared types incl. RecordingEntry
├── components/
│   ├── SetupScreen.tsx              ← NEW — onboarding screen (replaces ApiKeyGate)
│   ├── ApiKeyGate.tsx               ← OLD — still in codebase, safe to delete
│   ├── PhaseListening.tsx           ← largest file (1254 lines) — needs refactor
│   ├── PhaseMarkEntry.tsx
│   ├── PhaseGlobal.tsx
│   ├── PhaseReady.tsx
│   ├── PhaseReview.tsx
│   ├── PhaseSelect.tsx
│   ├── SpotifyPlayer.tsx
│   ├── HowToUse.tsx
│   └── LintPanel.tsx
├── hooks/
│   ├── useAnnotationState.ts
│   ├── useAudioDevices.ts
│   ├── useKeyboardShortcuts.ts
│   ├── useSpotifyPlayer.ts
│   └── useTimer.ts
└── lib/
    ├── excelExport.ts
    ├── lintAnnotation.ts
    ├── polishText.ts                ← uses OpenAI API for transcript polish
    ├── schema.ts
    ├── spotifyApi.ts
    └── spotifyAuth.ts               ← REDIRECT_URI needs updating (see below)
```

---

## CSS Variables (from index.css — use these, not custom ones)

```css
--bg, --surface, --surface-raised
--border, --border-active
--amber, --amber-bg, --amber-glow
--text, --text-muted, --text-dim
--error, --error-bg
--success
--font-mono, --font-serif, --font-display
--radius, --radius-pill
--transition
```

## Button Classes (from index.css — use these)
```
btn-primary, btn-ghost, btn-small, label
```

---

## What Was Built — Session 1 (Claude Code)

### Audio Recording + Dictation
- `RecordingEntry` type in `src/types/index.ts`
- `App.tsx` — recordings state + `addRecording`, `deleteRecording`, `clearRecordings`
- `PhaseListening.tsx` extended with:
  - Parallel MediaRecorder + Web Speech API on shared mic stream
  - Mic level meter (20-bar canvas, ~15fps)
  - Static waveform per saved recording (60-bar SVG)
  - `<audio controls>` player per recording
  - USE TRANSCRIPT / DOWNLOAD / DELETE per card
  - Whisper transcription via OpenAI API with inline key prompt
  - Collapsible RECORDINGS panel, auto-expands on new entry
  - Save to folder (File System Access API)
  - DELETE ALL TRACK / DELETE SESSION with save/discard confirmation
  - `beforeunload` warning when unsaved recordings exist
  - FINALIZING RECORDING… → AUDIO SAVED ✓ states
  - Live transcript display with 5s no-speech hint
  - Non-fatal transcript failure (audio saved even if no transcript)
  - Session-only notice at top of recordings panel

### Launch Stabilisation
- `vite.config.ts` — server host/port/strictPort
- `package.json` — `dev:chrome` script
- `scripts/open-chrome.sh` — created and executable

---

## What Was Built — Session 2 (Manual / This Session)

### SetupScreen (completed, committed)
Replaced `ApiKeyGate.tsx` with a full onboarding screen at `src/components/SetupScreen.tsx`:
- Hero / about section
- **01 / Spotify API Key** — required, saved to `sessionStorage('spotify_api_key')`
- **02 / OpenAI API Key** — optional, saved to `sessionStorage('openai_api_key')`
- **03 / Audio Input & Output** — mic + speaker dropdowns, 3-second mic test with level meter
- **04 / Browser Compatibility** — runtime detection of 4 APIs with ✓/✗ badges
- **ENTER APP** button — always enabled; warning shown if no Spotify key (user can still enter and control Spotify manually)

### App.tsx changes (completed, committed)
- Imports `SetupScreen` instead of `ApiKeyGate`
- Renders `<SetupScreen onEnter={handleApiKeyDone} />`
- SETUP button added next to Help (?) button — calls `setApiKeyDone(false)` to return to setup without clearing session keys

### Pending
- `src/lib/spotifyAuth.ts` — REDIRECT_URI still needs updating:
  ```ts
  const REDIRECT_URI = window.location.origin + '/callback'
  ```
- `ApiKeyGate.tsx` — safe to delete (no longer imported)

---

## Spotify Developer Setup

**Dashboard:** https://developer.spotify.com/dashboard
**App:** BeatPulse (existing — do NOT delete, 24hr creation limit in effect until ~9pm tonight)

**APIs to enable:**
- ✅ Web API
- ✅ Web Playback SDK
- ❌ Android SDK — off
- ❌ iOS SDK — off
- ❌ App Remote SDK — off

**Redirect URIs to add:**
```
https://beatpulselab.devknowsdev.workers.dev/callback
http://localhost:5173/callback
```

**Important:** App is in Development Mode. Add your Spotify account email as a test user (Users and Access section) before testing the live deployment. For public access, submit a quota extension request.

**Note:** Spotify app creation limit resets at ~9pm tonight (21 Feb). The existing app is fine to keep — no need to recreate it.

---

## Cloudflare Deployment

**Live URL:** https://beatpulselab.devknowsdev.workers.dev/
**Config:** `wrangler.jsonc` in project root
**Deploy:** automatic on push to `main`

```json
{
  "name": "beatpulse-annotator",
  "compatibility_date": "2026-02-21",
  "assets": {
    "directory": "./dist"
  }
}
```

---

## Planned: Flow Mode (not yet built — awaiting architecture decisions)

### What it is
A stripped-back annotation mode for uninterrupted listening. Designed for music supervisors who want to listen through a track without stopping to fill in forms.

### UI elements confirmed
- Track progress bar (scrubable)
- **⏮ -10s** / **⏭ +10s** skip buttons (not 15s)
- Play / Pause
- **Live audio level meter** — mic monitoring
- Tag buttons — silent stamp at current timestamp, no modal
- **Smart Dictate toggle** — continuous listening, auto-stamps transcript + timestamp as timeline notes, keyword routing to annotation fields
- **Speech to Text toggle** — live transcript visible on screen as spoken
- Timeline — read-only, live-updating as entries accumulate

### What is NOT in Flow Mode
- No "mark this moment" button/modal
- No global ratings panel
- No recording panel

### Architecture decisions still needed (ask user)
1. **New phase or toggle?**
   - A) Toggle within existing `listening` phase
   - B) Brand new `flow` phase in App.tsx phase router ← recommended, cleaner

2. **Tag tap behaviour:**
   - A) Silent stamp, no interruption ← true "flow"
   - B) Brief slide-up form, auto-dismisses after a few seconds

### Smart Dictate logic (first pass)
- Continuous SpeechRecognition (not one-shot)
- On each result: capture transcript + current track timestamp
- Save as timeline entry with category "Note"
- Optional: keyword matching to route to specific annotation fields
- No confirmation step — saves and continues

---

## Planned Refactor — PhaseListening.tsx (Claude Code)

`PhaseListening.tsx` at 1254 lines needs splitting. Run prompts one at a time, `npx tsc --noEmit` clean after each.

| Prompt | Action |
|--------|--------|
| 1 | Housekeeping — `.gitignore` DS_Store, delete `netlify.toml`, delete `ApiKeyGate.tsx` |
| 2 | Extract `src/hooks/useMicMeter.ts` |
| 3 | Extract `src/hooks/useAudioRecorder.ts` |
| 4 | Extract `src/hooks/useDictation.ts` |
| 5 | Extract `src/components/RecordingsPanel.tsx` |
| 6 | Final cleanup — PhaseListening < 300 lines, barrel file `src/hooks/index.ts` |
| 7 | Commit and push |

---

## Useful Terminal Commands

```bash
# Navigate to project
cd "/Users/duif/DK APP DEV/BeatPulseLab/beatpulse-annotator"

# Run locally (always use localhost, not 127.0.0.1)
npm run dev
# → open http://localhost:5173

# TypeScript check
npx tsc --noEmit

# Build for production
npm run build

# Commit and push
git add .
git commit -m "your message"
git push

# Check recent commits
git log --oneline -5

# Check file line counts
wc -l src/**/*.{ts,tsx,css} src/*.{ts,tsx,css} | sort -rn

# Check folder sizes
du -sh src/* | sort -rh
```

---

## Session Keys (sessionStorage — cleared on tab close)

| Key | Purpose |
|-----|---------|
| `spotify_api_key` | Spotify client token |
| `openai_api_key` | OpenAI key for Whisper + text polish |
| `beatpulse_api_key_gate_done` | Whether setup screen has been completed |

---

## Known Issues / Notes

- `PhaseListening.tsx` is 1254 lines — functional but needs refactor
- `netlify.toml` still in repo — safe to delete
- `ApiKeyGate.tsx` still in repo — safe to delete (SetupScreen replaced it)
- `.DS_Store` in `src/` — add to `.gitignore` and delete
- `spotifyAuth.ts` REDIRECT_URI still hardcoded — needs dynamic `window.location.origin + '/callback'`
- Build warning: JS bundle 1.17MB — will improve after refactor
- Web Speech API requires `localhost` or HTTPS — never use `127.0.0.1` for local dev
- OpenAI "AI Style Clean-Up" label in UI is vague — rename to "Whisper Transcription" during refactor

---

## Tools & Accounts

| Tool | Role |
|------|------|
| Cursor | Primary code editor — use for manual file edits |
| Claude Code | Terminal commands, TypeScript checks, multi-file refactors |
| Claude (chat) | Project management, writing code files, handover docs |
| GPT Plus | Parallel tasks, feasibility research |
| GitHub | Source control, triggers Cloudflare deploy on push |
| Cloudflare Pages | Hosting via Wrangler |
| Spotify Developer | beatpulse app — Web API + Web Playback SDK only |
