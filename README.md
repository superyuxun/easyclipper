# EasyClipper — Developer Context

> This file is for AI-assisted development context. Place it at the repo root as `CLAUDE.md`.

## What This App Does

EasyClipper is a **browser-based audio transcription and clipping tool**. The full pipeline:

1. User uploads an audio file
2. The app transcribes it in-browser using `@remotion/whisper-web` (no server required)
3. The transcript is displayed as individually-timestamped words
4. User clicks two words to select a range → the corresponding audio segment is extracted as a WAV subclip
5. The subclip can be trimmed and speed-adjusted before download

Everything runs client-side. There is no backend.

---

## Tech Stack

| Layer | Tech |
|---|---|
| Framework | React 19 + TypeScript |
| Build | Vite 7 |
| UI components | shadcn/ui (Radix UI primitives) + Tailwind CSS v3 |
| Transcription | `@remotion/whisper-web` 4.x — runs Whisper model in-browser via WASM |
| Audio processing | Web Audio API (`AudioContext`, `AudioBuffer`) — all custom, no library |
| Persistence | `localforage` — caches transcripts in IndexedDB by file hash |
| Virtual scroll | `react-virtuoso` — renders large transcripts efficiently |
| Routing/state | `use-query-params` — page state in URL query params |
| Package manager | Bun (lockfile present) — also works with npm |

---

## Project Structure

```
src/
├── App.tsx                          # Single large component — all app state lives here
├── components/
│   └── screens/
│       ├── upload-screen.tsx        # File drop/upload UI (react-dropzone)
│       ├── processing-screen.tsx    # Transcription progress UI, runs Whisper
│       └── edit-screen.tsx          # (currently empty/stub)
│   └── ui/                          # shadcn/ui components (button, card, form, input, label)
├── lib/
│   ├── audio-utils.ts               # Core audio primitives (see below)
│   ├── hash-file.ts                 # SHA-based file hashing for cache keys
│   ├── time.ts                      # formatTime() display helper
│   └── utils.ts                     # cn() Tailwind merge utility
├── types/
│   └── transcript.ts                # Caption and CachedTranscript interfaces
└── main.tsx
```

---

## Core Data Types

```typescript
// A single transcribed word with millisecond timestamps
interface Caption {
  text: string
  startMs: number
  endMs: number
  confidence: number | null
}

// Stored in localforage keyed by file hash
interface CachedTranscript {
  hash: string
  fileName: string
  fileSize: number
  processedAt: string
  numChunks: number
  chunkingEnabled: boolean
  modelUsed: string
  captions: Caption[]  // word-level, full transcript
}
```

---

## App State (App.tsx)

Key state variables and what they represent:

```typescript
// Routing
currentPage: 'upload' | 'processing' | 'edit'   // driven by ?page= query param
currentHash: string | null                         // file hash in ?hash= query param

// Audio
audioBuffer: AudioBuffer | null    // decoded PCM for the loaded file
audioUrl: string                   // object URL for <audio> element playback

// Transcript
captionsData: Caption[]            // all words, in order
chunks: ChunkGroup[]               // sentences (400–1200 chars), for virtual scroll rows

// Selection (single range, ephemeral)
selectionMode: boolean             // true after first word click
selectedRange: { start: number, end: number } | null  // word indices into captionsData

// Subclip (derived from selectedRange)
subclipUrl: string                 // object URL of the rendered WAV
subclipDuration: number            // seconds, reflects transforms

// Transform pipeline (applied in order: speed → trim)
transforms: Transform[]            // SpeedXform | TrimXform
speedEdits: SpeedEdit[]            // persistent speed segments within a subclip
trimValues: [number, number]       // [0, 100] percentage range
```

---

## Audio Pipeline (`src/lib/audio-utils.ts`)

Four exported functions — these are the building blocks for all audio work:

```typescript
getAudioContext(): AudioContext
// Singleton. Always use this — never construct AudioContext directly.

loadAudioBuffer(file: File): Promise<AudioBuffer>
// Decodes any browser-supported audio format to raw PCM.

trimAudioBuffer(buffer: AudioBuffer, startTime: number, endTime: number): AudioBuffer
// Extracts a time range (in seconds) from a buffer. Returns new AudioBuffer.
// Used for: creating subclips from selected word ranges.

audioBufferToWav(buffer: AudioBuffer): Promise<Blob>
// Encodes an AudioBuffer to 16-bit PCM WAV. Returns downloadable Blob.
// Used for: exporting subclips.
```

### Speed processing (inside App.tsx, not yet extracted)

- `applyMultipleSpeedSegments()` — applies multiple speed transforms to an AudioBuffer
- `timeStretchWSEW()` — WSOLA (Waveform Similarity Overlap-Add) time stretching, ~30ms window
- `mapTimeThroughMultipleSpeeds()` — maps a timestamp through speed-shifted segments

---

## Transcription Flow (`processing-screen.tsx`)

1. File → `loadAudioBuffer()` → `resampleTo16Khz()` (required by Whisper)
2. Optionally split into N chunks for progress reporting
3. `downloadWhisperModel()` + `transcribe()` from `@remotion/whisper-web`
4. `toCaptions()` converts Whisper output to `Caption[]`
5. Result saved to `localforage` under file hash
6. `onComplete(fileHash, cacheData, totalTime, audioBuffer)` called → navigates to edit page

---

## Transform Pipeline

When a subclip is selected, transforms are applied reactively in a `useEffect`:

```
audioBuffer
  → trimAudioBuffer(baseStart, baseEnd)   // extract selected word range
  → applyMultipleSpeedSegments(speeds)    // apply any speed edits
  → trimAudioBuffer(trim%)               // apply fine trim slider
  → audioBufferToWav()                   // encode to WAV
  → setSubclipUrl(objectURL)             // update preview player
```

Transforms array format:
```typescript
type SpeedXform = { kind: 'speed'; startSec: number; endSec: number; rate: number }
type TrimXform  = { kind: 'trim';  startPct: number; endPct: number }
type Transform  = SpeedXform | TrimXform
```

---

## Caching Strategy

- File identity = SHA hash (from `src/lib/hash-file.ts`)
- Transcript cached in `localforage` (IndexedDB) under that hash
- On upload, hash is checked first — if cached, skips transcription
- `AudioBuffer` is NOT cached (too large) — must be reloaded from file each session
- Cache cleared via "Clear cache & restart" button → `localforage.removeItem(hash)`

---

## Current Limitations / Known Tech Debt

- **`App.tsx` is a monolith** (~900 lines). All state, all handlers, all render logic in one component. No custom hooks extracted yet.
- **`edit-screen.tsx` is a stub** — the edit UI is entirely inside `App.tsx`, not in this file.
- **Single selection only** — `selectedRange` is replaced on each new selection. No multi-snippet accumulation.
- **No cross-file support** — only one `audioBuffer` in state at a time.
- **Speed processing is not extracted** — `applyMultipleSpeedSegments`, `timeStretchWSEW`, and `mapTimeThroughMultipleSpeeds` are defined inline in `App.tsx` rather than in `audio-utils.ts`.
- **`soundtouchjs`** is in dependencies but commented out — was an attempted alternative to the custom WSOLA implementation.

---

## Planned Feature: Multi-Snippet Stitching

The goal is to allow saving multiple snippets from one (or more) audio files and stitching them into a single exported WAV.

### New data model needed

```typescript
interface Snippet {
  id: string
  label: string
  sourceFileHash: string
  startMs: number
  endMs: number
  // transforms?: Transform[]  // optional: carry over speed/trim
}
```

### Implementation approach

1. **Snippet collection** — Add `snippets: Snippet[]` state. "Save snippet" button pushes current `selectedRange` (converted to ms) into the list.
2. **Snippet manager UI** — Ordered list with reorder, delete, preview per snippet.
3. **Stitch function** — Loop snippets → `trimAudioBuffer` each → concatenate `Float32Array` channel data → single `audioBufferToWav`. The primitives already exist.
4. **Cross-file support** — Replace `audioBuffer: AudioBuffer | null` with `audioBuffers: Map<fileHash, AudioBuffer>`. Snippets reference `sourceFileHash`.

### Stitch algorithm (pseudocode)

```typescript
async function stitchSnippets(snippets: Snippet[], buffers: Map<string, AudioBuffer>): Promise<Blob> {
  const ctx = getAudioContext()
  const sampleRate = buffers.values().next().value.sampleRate
  const trimmed = snippets.map(s => {
    const buf = buffers.get(s.sourceFileHash)!
    return trimAudioBuffer(buf, s.startMs / 1000, s.endMs / 1000)
  })
  const totalLength = trimmed.reduce((n, b) => n + b.length, 0)
  const out = ctx.createBuffer(2, totalLength, sampleRate)
  let offset = 0
  for (const buf of trimmed) {
    for (let ch = 0; ch < out.numberOfChannels; ch++) {
      out.getChannelData(ch).set(buf.getChannelData(ch), offset)
    }
    offset += buf.length
  }
  return audioBufferToWav(out)
}
```

### Recommended refactor before building

Extract these before adding new features — Claude Code performs better on smaller files:
- `src/hooks/useSnippets.ts` — snippet CRUD state
- `src/hooks/useAudioBuffers.ts` — multi-file buffer map
- `src/lib/audio-stitch.ts` — stitch + speed functions (move from App.tsx)
- `src/components/SnippetManager.tsx` — snippet list UI

---

## Development Setup

```bash
bun install       # or: npm install
bun run dev       # or: npm run dev
# App runs at http://localhost:5173
```

Whisper model is downloaded on first use (~150MB). Requires a browser with SharedArrayBuffer support (HTTPS or localhost).

---

## Key Dependencies to Know

| Package | Purpose |
|---|---|
| `@remotion/whisper-web` | In-browser Whisper transcription via WASM |
| `localforage` | IndexedDB abstraction for transcript caching |
| `react-virtuoso` | Virtual scroll for large transcripts |
| `react-slider` | Dual-thumb trim slider in detail view |
| `use-query-params` | URL-based page/hash routing |
| `react-dropzone` | File upload drag-and-drop |

---

# React + TypeScript + Vite

This template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react) uses [Babel](https://babeljs.io/) for Fast Refresh
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/) for Fast Refresh

## Expanding the ESLint configuration

If you are developing a production application, we recommend updating the configuration to enable type-aware lint rules:

```js
export default tseslint.config([
  globalIgnores(['dist']),
  {
    files: ['**/*.{ts,tsx}'],
    extends: [
      // Other configs...

      // Remove tseslint.configs.recommended and replace with this
      ...tseslint.configs.recommendedTypeChecked,
      // Alternatively, use this for stricter rules
      ...tseslint.configs.strictTypeChecked,
      // Optionally, add this for stylistic rules
      ...tseslint.configs.stylisticTypeChecked,

      // Other configs...
    ],
    languageOptions: {
      parserOptions: {
        project: ['./tsconfig.node.json', './tsconfig.app.json'],
        tsconfigRootDir: import.meta.dirname,
      },
      // other options...
    },
  },
])
```

You can also install [eslint-plugin-react-x](https://github.com/Rel1cx/eslint-react/tree/main/packages/plugins/eslint-plugin-react-x) and [eslint-plugin-react-dom](https://github.com/Rel1cx/eslint-react/tree/main/packages/plugins/eslint-plugin-react-dom) for React-specific lint rules:

```js
// eslint.config.js
import reactX from 'eslint-plugin-react-x'
import reactDom from 'eslint-plugin-react-dom'

export default tseslint.config([
  globalIgnores(['dist']),
  {
    files: ['**/*.{ts,tsx}'],
    extends: [
      // Other configs...
      // Enable lint rules for React
      reactX.configs['recommended-typescript'],
      // Enable lint rules for React DOM
      reactDom.configs.recommended,
    ],
    languageOptions: {
      parserOptions: {
        project: ['./tsconfig.node.json', './tsconfig.app.json'],
        tsconfigRootDir: import.meta.dirname,
      },
      // other options...
    },
  },
])
```
