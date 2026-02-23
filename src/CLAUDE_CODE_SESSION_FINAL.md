# TuneTag Annotator — Claude Code Master Session Doc
*Updated: 21 February 2026 — Session 3 (Consolidated)*
*Paste prompts ONE AT A TIME. Wait for TypeScript clean ✓ before moving to next.*

---

## ALREADY DONE — DO NOT REDO

### Session 1 (Claude Code)
- Audio recording + dictation in `PhaseListening.tsx`
- `RecordingEntry` type, `addRecording` / `deleteRecording` / `clearRecordings` in `App.tsx`
- Mic level meter, waveform SVG, Whisper transcription, collapsible recordings panel
- `vite.config.ts`, `package.json` dev scripts, `scripts/open-chrome.sh`

### Session 2 (Manual)
- `SetupScreen.tsx` replacing `ApiKeyGate.tsx`
- `App.tsx` updated: imports SetupScreen, SETUP button added

### Session 3 (Manual)
- `src/types/index.ts` — TagType, TagDef, TagPack, PhraseEntry, PromptsTagsLibraryState, UndoAction, TagPackImport; 'prompts_tags' added to Phase union; promptsTagsLibrary + undoStack added to AppState
- `src/lib/tagPacks.ts` — seed data (General, DnB, House, Trap) — NEW FILE
- `src/lib/tagLibrary.ts` — filter/group utilities — NEW FILE
- `src/lib/tagImport.ts` — JSON pack parser/validator — NEW FILE
- `src/lib/phraseBuilder.ts` — Who/What/Where/When sentence generator — NEW FILE
- `src/hooks/useAnnotationState.ts` — library state, undo stack, all library actions
- `src/components/PhasePromptsTags.tsx` — management screen (tabbed) — NEW FILE
- `src/components/PhaseMarkEntry.tsx` — reordered layout, collapsible sections, structured tag chips, phrase builder
- `src/components/PhaseSelect.tsx` — Prompts & Tags button added
- `src/App.tsx` — PhasePromptsTags wired in, library prop passed to PhaseMarkEntry
- `src/index.css` — all new styles appended (tag chips, collapsible panels, pack cards, phrase builder, import dropzone, undo toast)

---

## PRE-FLIGHT CHECK
*Run this first, before any prompt. Confirm all four outputs before proceeding.*

```
cd "/Users/duif/DK APP DEV/TuneTag/tunetag-annotator"
npx tsc --noEmit
git log --oneline -5
wc -l src/components/PhaseListening.tsx src/components/PhaseMarkEntry.tsx
```

Report: TypeScript result, last 5 commits, line counts of both files. Then wait for the first prompt.

---

## PHASE 0 — FIXES & SMALL ITEMS
*Do these before any new features. Fast, low-risk.*

---

### PROMPT 0A — Fix spotifyAuth REDIRECT_URI

```
For the project at /Users/duif/DK APP DEV/TuneTag/tunetag-annotator

In src/lib/spotifyAuth.ts, find the REDIRECT_URI constant and replace it with:
  const REDIRECT_URI = window.location.origin + '/callback'

Remove any hardcoded URL. Do not change anything else in the file.

Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 0B — Fix navigation routing bug

```
Diagnose and fix a navigation routing bug in the TuneTag Annotator project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

BUG: When a track is in_progress and the user navigates to 'prompts_tags',
clicking back does not reliably return them to 'select'. After a track is
started, navigation between phases feels broken.

STEP 1 — DIAGNOSE first. Read these files before writing any code:
  - src/components/PhasePromptsTags.tsx  (find the back/onBack handler)
  - src/components/PhaseSelect.tsx       (find what happens when an in_progress
                                          track card is clicked)
  - src/App.tsx                          (check any useEffect or conditional renders
                                          that change phase based on status or activeTrackId)

Report what you find in each before making any changes.

STEP 2 — FIX. Apply the minimal change needed so that:

  a) The back button in PhasePromptsTags always returns to 'select' — never
     dependent on activeTrackId or annotation status.

  b) In PhaseSelect, clicking an in_progress track routes to phase 'ready',
     not straight to 'listening' or 'prompts_tags'. The user must always
     land on PhaseReady first so they can confirm before resuming.

  c) No phase auto-redirects to another based on status alone. Phase changes
     happen only on explicit user action. Remove any useEffect or conditional
     render that silently reroutes the user.

STEP 3 — VERIFY. Trace these paths out loud based on your changes:
  PhaseSelect → click in_progress track → PhaseReady → START → PhaseListening
  PhaseListening → PROMPTS & TAGS → PhasePromptsTags → BACK → PhaseSelect
  Confirm each step routes correctly.

Run npx tsc --noEmit. Report TypeScript clean ✓
Do not change annotation logic, timer logic, or UI layout. Routing only.
```

---

### PROMPT 0C — Collapsible tag sections default closed

```
For the project at /Users/duif/DK APP DEV/TuneTag/tunetag-annotator

In src/components/PhaseMarkEntry.tsx:
  Find all useState calls that control collapsible open/closed state for
  tag sections and section-type panels. Change their default from true to false
  so all collapsible sections start closed.

In src/components/PhasePromptsTags.tsx:
  Do the same — any collapsible sections that default open should default closed.

Do not change any logic, routing, or layout. Only the boolean default values.

Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

## PHASE 1 — RESEARCHED TAG PACKS
*Import the AI-researched genre vocabulary into the app.*

---

### PROMPT 1A — Save researched tag pack JSON as static assets

```
For the project at /Users/duif/DK APP DEV/TuneTag/tunetag-annotator

The app now has AI-researched genre tag packs in JSON format (schema matches
the TagPackImport type in src/types/index.ts and the parser in src/lib/tagImport.ts).
The research is delivered in 4 parts. We are starting with Part 1.

1. Create the file public/tag-packs-part1.json and paste in the JSON content
   I am about to provide. Do not modify the JSON content at all.

2. Verify the file is accessible at runtime by checking it would be served
   from the Vite public directory (files in /public are served at the root path).

3. Do NOT import or use this file yet — that comes in the next prompt.

4. Run: cat public/tag-packs-part1.json | python3 -m json.tool > /dev/null
   (This validates JSON syntax.) Report: valid JSON ✓ or the parse error.

PASTE THE JSON NOW:
[[ USER: paste the full Part 1 JSON here — general, pop, rock_indie, hiphop_trap packs ]]
```

> **Note:** Repeat PROMPT 1A for each additional part (1B, 1C, 1D) as GPT/Gemini deliver them, saving as `tag-packs-part2.json`, `tag-packs-part3.json` etc.

---

### PROMPT 1B — Auto-load researched packs on app startup

```
For the project at /Users/duif/DK APP DEV/TuneTag/tunetag-annotator

We have researched tag pack JSON files saved in /public (tag-packs-part1.json,
and eventually part2, part3, part4). We need the app to fetch and merge these
into the library state on startup.

The existing import infrastructure is in src/lib/tagImport.ts (parseTagPackImport
function) and the library state lives in src/hooks/useAnnotationState.ts
(importTagPack action).

1. Create src/lib/loadResearchedPacks.ts:

   import { parseTagPackImport } from './tagImport'
   import type { PromptsTagsLibraryState } from '../types'

   const PACK_FILES = [
     '/tag-packs-part1.json',
     '/tag-packs-part2.json',
     '/tag-packs-part3.json',
     '/tag-packs-part4.json',
   ]

   export async function loadResearchedPacks(
     importTagPack: (raw: unknown) => void
   ): Promise<void> {
     for (const url of PACK_FILES) {
       try {
         const res = await fetch(url)
         if (!res.ok) continue   // file doesn't exist yet — skip silently
         const raw = await res.json()
         // The JSON wraps packs in a top-level "packs" array
         // importTagPack expects one pack at a time
         if (raw?.packs && Array.isArray(raw.packs)) {
           for (const pack of raw.packs) {
             importTagPack(pack)
           }
         }
       } catch {
         // Network or parse error — skip silently, do not block app load
       }
     }
   }

2. In src/App.tsx:
   - Import loadResearchedPacks from './lib/loadResearchedPacks'
   - Add a useEffect that runs once on mount (after apiKeyDone is true):
     useEffect(() => {
       if (!apiKeyDone) return
       loadResearchedPacks(state.importTagPack)
     }, [apiKeyDone])

   This runs after setup is complete and silently loads whatever pack files exist.
   Missing files are skipped without error.

3. The existing hardcoded seed packs in tagPacks.ts remain untouched.
   The researched packs are additive — they merge in via the existing importTagPack
   action which deduplicates by normalized label.

4. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 1C — Commit Phase 1 (tag packs + fixes)

```
Commit and push all Phase 0 and Phase 1 work for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. git add .
2. git commit -m "Add researched genre tag packs, auto-loader, routing fixes, collapsible defaults"
3. git push
4. Confirm success. Report git log --oneline -5.
```

---

## PHASE 2 — NEW FEATURES

---

### PROMPT 2A — Wire Flow Mode into App.tsx

```
For the project at /Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. In src/types/index.ts, add 'flow' to the Phase union type.
   (The union already has 'prompts_tags' — add 'flow' alongside it.)

2. In src/App.tsx:
   - Add import: import { PhaseFlow } from './components/PhaseFlow'
   - Add to the phase router after the listening/mark_entry block:

     {phase === 'flow' && activeAnnotation && (
       <PhaseFlow
         annotation={activeAnnotation}
         elapsedSeconds={timer.elapsedSeconds}
         isTimerRunning={timer.isRunning}
         timerStart={timerStart}
         timerPause={timerPause}
         setPhase={state.setPhase}
         updateTimeline={state.updateTimeline}
         setStatus={state.setStatus}
         spotifyToken={spotifyToken}
         spotifyPlayer={spotifyPlayer}
       />
     )}

3. Create a typed placeholder at src/components/PhaseFlow.tsx:

   import type { TrackAnnotation, Phase, TimelineEntry } from '../types'

   interface Props {
     annotation: TrackAnnotation
     elapsedSeconds: number
     isTimerRunning: boolean
     timerStart: () => void
     timerPause: () => void
     setPhase: (phase: Phase) => void
     updateTimeline: (trackId: number, entries: TimelineEntry[]) => void
     setStatus: (trackId: number, status: TrackAnnotation['status'], extra?: Partial<TrackAnnotation>) => void
     spotifyToken: string | null
     spotifyPlayer: any
   }

   export function PhaseFlow({ setPhase }: Props) {
     return (
       <div style={{ padding: '2rem', color: 'var(--amber)', fontFamily: 'var(--font-mono)' }}>
         FLOW MODE — coming soon
         <br /><br />
         <button onClick={() => setPhase('listening')}>← EXIT</button>
       </div>
     )
   }

4. In PhaseListening.tsx, find the top controls area and add a FLOW MODE button
   that calls setPhase('flow'). Style to match existing buttons. Label: "⟩ FLOW MODE"

5. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 2B — Build PhaseFlow.tsx

```
Replace the placeholder src/components/PhaseFlow.tsx for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

PROPS (already wired from App.tsx):
  annotation, elapsedSeconds, isTimerRunning, timerStart, timerPause,
  setPhase, updateTimeline, setStatus, spotifyToken, spotifyPlayer

LAYOUT — full screen, immersive, no scrolling:

TOP BAR (fixed, full width)
- Left: track name + artist (truncated)
- Centre: elapsed mm:ss — large amber monospace
- Right: EXIT FLOW MODE button → setPhase('listening')

PROGRESS BAR (full width, below top bar)
- Fill = elapsedSeconds / (annotation.track.durationSeconds ?? 300)
- Background: var(--surface), fill: var(--amber)

TRANSPORT ROW (centred, 44px min touch targets)
- ⏮ -10s | ⏸/▶ Play/Pause (large, btn-primary) | +10s ⏭
- Play/Pause calls timerStart() / timerPause()

MIC LEVEL METER
- getUserMedia({ audio: true }) on mount
- AudioContext + AnalyserNode, 20 bars, amber, ~15fps via rAF
- Full width, label "MIC" on left
- Release stream + close AudioContext on unmount

TAG BUTTONS
- Use the same category list as PhaseMarkEntry
- Large pill buttons, responsive grid: repeat(auto-fill, minmax(140px, 1fr))
- On tap: create TimelineEntry { id: crypto.randomUUID(), timestamp: formatMSS(elapsedSeconds),
  category: tagName, note: '', createdAt: Date.now() }
  Call updateTimeline(annotation.track.id, [...annotation.timeline, newEntry])
  Show 1.5s toast: "✓ [TagName]" — NO modal, NO interruption

SMART DICTATE TOGGLE ("● SMART DICTATE")
- When ON: continuous SpeechRecognition, each final result creates a
  TimelineEntry (category: 'Note', note: transcript), shown briefly on screen (fades 3s)
- When OFF: stop recognition
- Hide entirely if SpeechRecognition unsupported

SPEECH TO TEXT TOGGLE ("◎ SPEECH TO TEXT")
- When ON: show live transcript box (interim = muted, final = full colour, fades 4s)
- Independent of Smart Dictate — both can be on simultaneously

TIMELINE DRAWER
- Fixed bottom toggle "TIMELINE (N)" — slides up as drawer
- Read-only, newest first: timestamp | category | note (truncated)
- Close button dismisses

HELPER FUNCTION (copy locally):
  function formatMSS(s: number) {
    return `${Math.floor(s/60)}:${String(s%60).padStart(2,'0')}`
  }

STYLING: var(--bg), var(--font-mono) for labels/times, var(--font-serif) for content,
var(--amber) accent, generous spacing.

ON UNMOUNT: stop SpeechRecognition, release mic stream, close AudioContext.

Run npx tsc --noEmit. Report TypeScript clean ✓ and line count of PhaseFlow.tsx.
```

---

### PROMPT 2C — Improve PhaseSelect layout

```
Improve the PhaseSelect.tsx layout for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

IMPORTANT: The file already has a "PROMPTS & TAGS" button. Do NOT remove it.
Do not change any routing logic. Visual layout only.

1. LAYOUT — responsive card grid:
   grid-template-columns: repeat(auto-fill, minmax(320px, 1fr))
   Max width: 900px centred. Cards: min-height 140px, more padding.

2. TRACK CARDS — add:
   - Elapsed time if > 0: ⏱ mm:ss (amber monospace, annotation.elapsedSeconds)
   - Timeline entry count if in_progress or complete: "N timeline entries"
   - Complete tracks: subtle amber bottom border or background tint
   - Status badge larger with coloured background:
       not_started: muted/dim
       in_progress: amber background
       complete: success green
       skipped: dim + strikethrough

3. HEADER — add:
   - "N of N tracks complete" subtitle in muted monospace
   - CONTINUE SESSION button if any tracks are in_progress
     (routes to the first in_progress track → 'ready' phase)

4. EMPTY STATE — if all tracks not_started:
   "Select a track below to begin annotating"

Use existing CSS vars. Keep all onClick logic as-is.
Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 2D — Global left sidebar

```
Add a global collapsible left sidebar to the TuneTag Annotator project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

The sidebar replaces the current floating SETUP, ?, and Connect Spotify buttons in App.tsx.
Visible on every phase EXCEPT 'flow'.

1. Create src/components/AppSidebar.tsx

   Props:
     phase: Phase
     activeAnnotation: TrackAnnotation | null
     timerElapsed: number
     timerRunning: boolean
     onSetup: () => void
     onHelp: () => void
     onSpotifyLogin: () => void
     spotifyToken: string | null
     spotifyPlayer: any

   TOGGLE TAB (always visible, even when sidebar closed):
   - Fixed, left: 0, top: 50%, transform: translateY(-50%)
   - Small pill: ▶ closed / ◀ open
   - Amber, monospace, z-index: 60

   SIDEBAR PANEL (slides in from left, width: 220px):
   - position: fixed, left: 0, top: 0, height: 100vh
   - Background: var(--surface), border-right: 1px solid var(--border)
   - CSS transition on transform (translateX(-100%) ↔ translateX(0))
   - z-index: 50

   CONTENTS (top to bottom):
   All items: font-family var(--font-mono), font-size 0.6875rem,
   letter-spacing 0.05em, padding 0.75rem 1rem. Buttons use btn-ghost class.

   APP section:
     [ SETUP ]   — onSetup()
     [ ? HELP ]  — onHelp()

   Divider (1px var(--border))

   SPOTIFY section:
     If no token: [ ♫ CONNECT SPOTIFY ] — onSpotifyLogin()
     If token: render <SpotifyPlayer> inline (compact)

   Divider

   SESSION section (only if activeAnnotation exists):
     ⏱ elapsed mm:ss — amber if running, dim if paused
     Track name (truncated, 1 line, muted)

   On desktop (>768px): no overlay, sidebar floats over content.
   On mobile: dim overlay behind sidebar, click to close.

2. In App.tsx:
   - Remove: the existing floating SETUP + ? buttons div
   - Remove: the floating Connect Spotify button
   - Remove: the SpotifyPlayer render from the top level
   - Import and render <AppSidebar /> just before the phase router
   - Hide when phase === 'flow' (phase === 'flow' ? null : <AppSidebar ... />)
   - Pass all required props

3. Run npx tsc --noEmit. Report TypeScript clean ✓
   Report line count of AppSidebar.tsx.
```

---

### PROMPT 2E — Per-track genre pack selection

```
Add per-track genre pack selection to the TuneTag Annotator project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. In src/types/index.ts:
   Add to TrackAnnotation:
     activePackIds?: string[]
   When undefined = use global library defaults.

2. In src/hooks/useAnnotationState.ts:
   Add action:
     setTrackPackIds(trackId: number, packIds: string[]): void
   Sets annotation[trackId].activePackIds = packIds and saves state.

3. In src/components/PhaseReady.tsx:
   Add props:
     library: PromptsTagsLibraryState
     onSetPackIds: (packIds: string[]) => void

   Add a "ACTIVE PACKS FOR THIS SESSION" section below the annotator name field:
   - Show all available packs as toggle chips (.tag-chip style)
   - Pre-select: annotation.activePackIds if set, else packs where enabled === true
   - Local state manages the selection
   - On START LISTENING: call onSetPackIds(selectedPackIds) THEN onStartListening()

4. In src/App.tsx:
   Pass to PhaseReady:
     library={state.promptsTagsLibrary}
     onSetPackIds={(ids) => state.setTrackPackIds(activeAnnotation!.track.id, ids)}

5. In src/components/PhaseMarkEntry.tsx:
   Update tag filtering so available tags use:
     annotation.activePackIds ?? packs where library.packs[id].enabled === true
   The annotation prop is already available.

6. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 2F — Commit Phase 2

```
Commit and push Phase 2 for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. git add .
2. git commit -m "Flow Mode, improved PhaseSelect, global sidebar, per-track pack selection"
3. git push
4. Confirm success. Report git log --oneline -8.
```

---

## PHASE 3 — FULL SCREEN LAYOUT
*Priority: medium. Do after Phase 2 is committed.*

---

### PROMPT 3A — Full Screen layout toggle in PhaseListening

```
Add a Full Screen layout mode toggle to PhaseListening.tsx for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

IMPORTANT: PhaseListening.tsx is large. Make surgical changes only.
Do NOT restructure, rename, or move any existing logic.

1. Add: const [viewMode, setViewMode] = useState<'classic' | 'fullscreen'>('classic')

2. Add toggle button next to FLOW MODE button:
   "⛶ FULL" when classic, "⊠ EXIT FULL" when fullscreen

3. When viewMode === 'classic': render existing layout UNCHANGED.

4. When viewMode === 'fullscreen':

   FIXED TOP BAR (full width)
   - Left: track name + artist
   - Centre: elapsed mm:ss, large amber monospace
   - Right: view toggle | FLOW MODE | DONE

   FULL WIDTH PROGRESS BAR below top bar

   TRANSPORT ROW (44px min targets)
   - ⏮ -10s | ⏸/▶ | +10s ⏭

   MIC METER (full width, visible only when recording active)

   TAG GRID — all existing mark-entry trigger buttons
   grid-template-columns: repeat(auto-fill, minmax(140px, 1fr))
   Large pill style

   FIXED BOTTOM TOOLBAR
   - Left: DICTATE (existing behaviour)
   - Centre: RECORDINGS toggle (shows count)
   - Right: TIMELINE toggle (shows entry count)

   TIMELINE DRAWER — slides up from bottom, existing content, close button
   RECORDINGS DRAWER — slides up from bottom, existing content, close button

5. Run npx tsc --noEmit. Report TypeScript clean ✓
   Report final line count of PhaseListening.tsx.
```

---

### PROMPT 3B — Commit Phase 3

```
Commit and push Phase 3 for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. git add .
2. git commit -m "Add Full Screen layout toggle to PhaseListening"
3. git push
4. Confirm success.
```

---

## PHASE 4 — REFACTOR
*Do LAST — only after all features are committed and working.*

---

### PROMPT 4A — Housekeeping

```
Clean up project housekeeping for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. Add to .gitignore if not present: .DS_Store and **/.DS_Store
2. Delete src/.DS_Store if it exists
3. Delete netlify.toml — project is on Cloudflare
4. Delete src/components/ApiKeyGate.tsx — replaced by SetupScreen
5. Review scripts/ folder:
   - Keep scripts/open-chrome.sh if still useful
   - Delete HOW_TO_RUN.md if it references Netlify or outdated setup
6. git add . && git status — report staged removals
7. npx tsc --noEmit — report TypeScript clean ✓
```

---

### PROMPT 4B — Extract useMicMeter hook

```
Refactor PhaseListening.tsx for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. Create src/hooks/useMicMeter.ts:
   - Accepts: stream: MediaStream | null
   - AudioContext + AnalyserNode, returns barLevels: number[] (20 values, 0–1)
   - Updates at ~15fps via requestAnimationFrame
   - Cleans up when stream is null or on unmount

2. Update PhaseFlow.tsx to use useMicMeter instead of inline AudioContext code.

3. Remove inline mic meter code from PhaseListening.tsx, replace with useMicMeter(micStream).

4. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 4C — Extract useAudioRecorder hook

```
Refactor PhaseListening.tsx for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. Create src/hooks/useAudioRecorder.ts exposing:
   - micStream: MediaStream | null
   - status: 'idle' | 'recording' | 'finalizing'
   - startRecording(): Promise<void>
   - stopRecording(): void
   - cancelRecording(): void
   - onRecordingReady callback: (blob: Blob, mimeType: string) => void

2. Remove inline MediaRecorder code from PhaseListening.tsx.
   Replace with useAudioRecorder().

3. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 4D — Extract useDictation hook

```
Refactor PhaseListening.tsx for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. Create src/hooks/useDictation.ts exposing:
   - startDictation(stream: MediaStream): void
   - stopDictation(): void
   - liveTranscript: string
   - finalTranscript: string
   - noSpeechHint: boolean (true after 5s silence)
   - reset(): void
   Hook does NOT manage the mic stream — receives it as parameter.

2. Remove inline SpeechRecognition code from PhaseListening.tsx.
   Replace with useDictation().

3. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 4E — Extract RecordingsPanel component

```
Refactor PhaseListening.tsx for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. Create src/components/RecordingsPanel.tsx with props:
   - recordings: RecordingEntry[]
   - isOpen: boolean
   - onToggle: () => void
   - onDelete: (id: string) => void
   - onDeleteAllTrack: (trackId: number) => void
   - onDeleteSession: () => void
   - onUseTranscript: (transcript: string) => void
   - currentTrackId: number

   Move ALL of these into RecordingsPanel:
   - Collapsible panel shell + toggle
   - Session-only warning notice
   - Per-recording cards (waveform SVG, audio player, transcript, action buttons)
   - Save/Discard confirmation dialog
   - DELETE ALL TRACK / DELETE SESSION / SAVE TO FOLDER buttons
   - Whisper transcription + inline API key input
   - beforeunload warning effect

2. Replace all of this in PhaseListening.tsx with: <RecordingsPanel ... />

3. Run npx tsc --noEmit. Report TypeScript clean ✓
```

---

### PROMPT 4F — Final cleanup and barrel file

```
Final cleanup for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. Check PhaseListening.tsx line count.
   If still over 300 lines, identify what can be further extracted and do so.

2. Create src/hooks/index.ts barrel file:
   export { useDictation } from './useDictation'
   export { useAudioRecorder } from './useAudioRecorder'
   export { useMicMeter } from './useMicMeter'
   export { useAnnotationState } from './useAnnotationState'
   export { useAudioDevices } from './useAudioDevices'
   export { useKeyboardShortcuts } from './useKeyboardShortcuts'
   export { useSpotifyPlayer } from './useSpotifyPlayer'
   export { useTimer } from './useTimer'

3. Update imports in any file that can use the barrel.

4. Run npx tsc --noEmit. Report TypeScript clean ✓

5. Report final line counts:
   wc -l src/**/*.{ts,tsx,css} src/*.{ts,tsx,css} | sort -rn
```

---

### PROMPT 4G — Final commit

```
Commit and push the full refactor for the project at
/Users/duif/DK APP DEV/TuneTag/tunetag-annotator

1. git add .
2. git commit -m "Refactor: extract useMicMeter, useAudioRecorder, useDictation, RecordingsPanel; housekeeping"
3. git push
4. Confirm success. Report git log --oneline -8.
```

---

## SUMMARY TABLE

| Phase | Prompts | Priority | Skip if short on compute? |
|-------|---------|----------|--------------------------|
| 0 — Fixes | 0A, 0B, 0C | Do first | No |
| 1 — Tag Packs | 1A (×4), 1B, 1C | High | No |
| 2 — New Features | 2A–2F | Highest | No — do all |
| 3 — Full Screen | 3A, 3B | Medium | Yes |
| 4 — Refactor | 4A–4G | Low | Yes |

**Stop after 2F if compute is low. Everything in Phases 0–2 is the priority.**

---

## QUICK REFERENCE

```bash
cd "/Users/duif/DK APP DEV/TuneTag/tunetag-annotator"
npm run dev            # always localhost:5173, never 127.0.0.1
npx tsc --noEmit       # TypeScript check
npm run build          # production build
git add . && git commit -m "msg" && git push
git log --oneline -5
wc -l src/**/*.{ts,tsx,css} src/*.{ts,tsx,css} | sort -rn
```

## CSS Variables
```css
--bg, --surface, --surface-raised
--border, --border-active
--amber, --amber-bg, --amber-glow
--text, --text-muted, --text-dim
--error, --error-bg, --success
--font-mono, --font-serif, --font-display
--radius, --radius-pill, --transition
```

## Button Classes
```
btn-primary  btn-ghost  btn-small  label
```

## Tag Pack JSON — How to Deliver Parts

The researched JSON (from GPT/Gemini) comes in up to 4 parts. Each part is a valid
JSON object with the top-level shape:
```json
{
  "version": 1,
  "packs": [ { "packId": "...", "terms": [...] } ],
  "globalNotes": { ... }
}
```
Save each part as `public/tag-packs-part1.json` through `tag-packs-part4.json`.
The auto-loader (Prompt 1B) fetches all four, skipping any that don't exist yet.
You can add parts incrementally — each deploy will pick up whatever files are present.
