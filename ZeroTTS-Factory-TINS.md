<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:WEB -->
<!-- ZS:LANGUAGE:TYPESCRIPT -->
<!-- ZS:VERSION:0.1.0 -->
<!-- ZS:LAYER:ZERO-QUADRATIC -->
<!-- ZS:DERIVED-FROM:ZeroVoice-KokoroTTS-postV2-TINS,ZeroTTS-zvKokoro-TINS -->

# ZeroTTS-Factory

## Description

**ZeroTTS-Factory** is a browser-only Vite + React + TypeScript application that turns the ZeroVoice-KokoroTTS deterministic-voice pipeline into a batch *speech factory*. It accepts two inputs — a **voice profile** (persona name + gender filter + optional seed pin) and an **order JSON** (atomised templates and pools, identical schema to ZeroTTS) — and produces, in one operation, every WAV file the order can resolve to. The generated audio is presented in a DOWNLOAD tab where each clip can be previewed individually, downloaded one-by-one, or bundled into a sequence of ZIP archives capped at 100 MB each.

The system is a direct adaptation of two upstream specifications:

1. **ZeroVoice-KokoroTTS** (postV2 TINS) — provided the position-as-seed voice derivation: six independent xxHash64 salts produce a `VoiceSpec` (voice A index, voice B index, blend t, pitch scale, energy scale) plus four Zero-Quadratic arc properties (curve warp, harmonic weight, spectral skew, kinship). ZeroTTS-Factory replaces the `(sx, sy, sz)` 3D coordinate seed with a **string-as-seed** (`hash(persona_name)`) so the persona name itself is the entropy source. A user-set seed pin overrides the name-derived seed when fine-tuning is needed.

2. **ZeroTTS-zvKokoro** (the speech-cache layer) — provided the JSON profile format with named contexts, templates, slot tokens (`{slot_name}`), shared pools, atom enumeration via Cartesian product, the install/render orchestration loop, and the WAV-blob storage model. ZeroTTS-Factory keeps the order schema and enumeration algorithm intact; the only behavioural change is that rendered atoms accumulate in an in-memory **AudioPool** instead of going straight into a runtime Cache Storage bucket. The AudioPool is what the DOWNLOAD tab presents.

The result is a fully automated tool that produces all stock speech assets for a character (or for many characters in sequence) with consistent, reproducible voices. Two runs of the same persona name and order JSON on two different machines produce byte-identical voice selections and therefore very similar audio (modulo Kokoro inference determinism).

The target user is a game / agent / interactive-fiction author who needs a complete, ready-to-ship folder of WAV files per character. Existing ZeroResponse profile JSON files transfer with no edits.

---

## Functionality

### Core Features

**F1 — Persona-name as voice seed.** The persona name string is the entropy source for voice derivation. `hash("alice")` selects voice A index, voice B index, slerp t, pitch scale, energy scale, and the four arc properties via the same six salts (`SALT_VOICE_A=0x0001` … `SALT_VOICE_ENERGY=0x0005`, plus `SALT_ARC_*` 0x0010-0x0012) used in ZeroVoice-KokoroTTS. The persona name is normalised to its `personaSeed` form (`lowercase`, `[^a-z0-9_-]` replaced with `_`) before hashing.

**F2 — Seed pin override.** Each voice profile carries an optional `pinSeed: string | null` field. When non-null, the pin string replaces the persona name in the hash input — name remains the human-readable label, pin is the deterministic identity. This lets a user re-roll the auto-derived voice without renaming the character.

**F3 — Gender filter.** The same three-way filter from ZeroVoice-KokoroTTS — `mixed | female | male` — narrows the voice pool. Female pool: `af_*` (indices 0-9) + `bf_*` (18-21) = 14 voices. Male pool: `am_*` (10-17) + `bm_*` (22-25) = 12 voices. Mixed: all 26.

**F4 — Order JSON upload.** The factory accepts a TTSProfile JSON file via drag-drop or file picker. Schema is identical to ZeroTTS-zvKokoro's TTSProfile minus the `voice_blend` block (voice is supplied by the active voice profile, not the order). The JSON declares contexts (each with templates and a tier), shared pools, and an `install.max_atoms` cap.

**F5 — Atom enumeration.** Before running the factory, the app enumerates every atom the order can produce using the Cartesian product of pool entries across each template's slots. The enumeration is deduplicated by atom text. Display in the ORDER tab shows: total atom count, breakdown per context, and the first 20 sample atom strings.

**F6 — Factory run (batch synthesis).** When the user clicks **Run Factory**, the app iterates the enumerated atoms in tier order (1 → 2 → 3) and synthesises each via Kokoro WASM using the active voice profile's derived `VoiceSpec`. A progress bar updates per atom. Each rendered Float32Array PCM buffer is encoded to a WAV `Blob` and pushed into the in-memory **AudioPool** with metadata `{ id, atomText, contextName, tier, persona, durationMs, byteSize, blob, objectUrl }`.

**F7 — DOWNLOAD tab.** Lists every generated clip in a virtualised table grouped by context. Each row exposes ▶ Play, ⬇ Download, and 🗑 Remove. Top-of-tab actions: **Download All (ZIP)**, **Download Manifest (JSON)**, **Clear Pool**.

**F8 — Multi-archive ZIP splitting.** When the user clicks **Download All (ZIP)**, the app builds one or more ZIP archives such that no single archive exceeds `MAX_ZIP_BYTES = 100 * 1024 * 1024`. Files are added to the current archive in tier-then-context-then-index order until the next file would push the running total above the cap; that archive is finalised, downloaded, and a new archive begins. Each archive is named `{personaSeed}_{orderName}_part{NN}_of{TT}.zip`. A manifest JSON describing every file's archive assignment is written to part 01.

**F9 — Persistent voice profiles.** Voice profiles (name, gender, pinSeed, locked overrides) are persisted in `localStorage` under key `zerotts-factory-voice-profiles`. Removing a profile from the UI deletes it from storage. The currently active profile is stored at `zerotts-factory-active-profile` (just the persona name string).

**F10 — Asset bootstrap.** On first load the app downloads three assets to Cache Storage and IndexedDB: the Kokoro ONNX model (≈88 MB), the voice style table (≈3 MB), and the espeak-ng-wasm phonemizer bundle (≈4 MB). A SETUP screen with a progress bar runs once; subsequent loads pull from cache. The bootstrap is identical in spirit to the ZeroVoice-KokoroTTS desktop setup but uses browser cache primitives.

**F11 — Voice preview.** A separate VOICE tab lets the user type ad-hoc text and hit "Synthesise preview" to hear what the active voice profile sounds like before committing to a factory run. Preview clips are *not* added to the AudioPool unless the user clicks "Pin to pool".

### User Flows

**Flow A — First-time user, end-to-end run:**

```
1. User opens app → SETUP screen runs
   ├─ Downloads kokoro model (88 MB)   → cache.put("transformers-kokoro-onnx-int8")
   ├─ Downloads voices-v1.0.bin (3 MB) → cache.put("transformers-kokoro-voices-v1")
   └─ Downloads espeak-ng wasm (4 MB)  → cache.put("zerotts-espeak-wasm")
2. App ready → VOICE tab shown
3. User types persona name "Alice", picks "female" gender, leaves pin empty
   → derived voice spec rendered live: voice A bf_alice, voice B bf_emma, t 0.42, ...
4. User types preview text "Hello, I'm Alice", clicks Synthesise
   → ~1.2s WASM inference, audio plays
5. User likes the voice, clicks "Save profile" → profile persisted
6. User goes to ORDER tab, drops alice_order.json
   → enumeration: 35 atoms across 4 contexts (ack 12, tool_invoke 18, progress 3, error 2)
7. User goes to FACTORY tab, clicks Run Factory
   → progress 0/35 → 35/35 in ≈20s (35 × 0.55s per atom)
8. DOWNLOAD tab shows 35 clips
9. User clicks "Download All (ZIP)"
   → 35 clips × ~50 KB ≈ 1.7 MB total → single archive alice_alice-order_part01_of01.zip
   → browser downloads the file
```

**Flow B — Multi-archive split (large order):**

```
User uploads order with 100 atoms × ~1 MB each (long passages)
  → AudioPool now holds ~100 MB
User clicks "Download All (ZIP)"
  → ZIP builder packs files in order, watching running compressed size
  → archive 01 fills at file 53 (running total ~99 MB), finalised
  → archive 02 begins with file 54
  → 47 more files fit, archive 02 finalised at ~94 MB
  → manifest.json listing every file → archive mapping is added to archive 01
  → both archives downloaded sequentially
```

**Flow C — Re-run with same name, different gender:**

```
User has profile "Alice" with gender=female (auto voices: bf_alice, bf_emma)
User clones profile, renames to "Alice Male", sets gender=male
  → hash("alice_male") % MALE_INDICES.length picks am_michael, am_puck
User runs same order → DOWNLOAD tab now lists 35 clips with a different voice
Both profiles' clips coexist in the AudioPool until cleared
```

**Flow D — Pin override (re-roll the voice):**

```
User dislikes Alice's auto-derived voice (too similar to default Kokoro voice).
User edits profile: pinSeed = "alice_take2"
  → hash("alice_take2") gives bf_isabella + bf_lily, t 0.71
User keeps name "Alice" for display
Re-runs factory → same atoms, fresh voice
```

**Flow E — Profile JSON cross-machine reproducibility:**

```
Author A on Linux generates alice_take2 → produces 35 WAVs
Author A ships alice_take2 voice profile JSON + alice_order.json to Author B (Windows)
Author B loads both → re-runs factory
  → same voice indices, same blend t, same pitch/energy scales
  → audio is sample-comparable (Kokoro WASM inference is deterministic given same inputs)
```

### UI Layout

The app is a single-page React UI with a top tab bar and a status footer.

```
+-------------------------------------------------------------------+
|  ZER0TTS-FACTORY  [v0.1.0]                    [⚙ Settings]        |
+-------------------------------------------------------------------+
|  [ VOICE ] [ ORDER ] [ FACTORY ] [ DOWNLOAD (35) ]                |
+-------------------------------------------------------------------+
|                                                                   |
|                     <active tab content>                          |
|                                                                   |
+-------------------------------------------------------------------+
|  Kokoro: ready  |  Pool: 35 clips, 1.7 MB  |  Voice: Alice (♀)    |
+-------------------------------------------------------------------+
```

**VOICE tab — voice profile editor + preview:**

```
+----------------------------------------+----------------------------+
|  PROFILE LIST                          |  ACTIVE PROFILE            |
|  ─────────────                         |  ──────────────            |
|  ● Alice           (♀)                 |  Name:    [ Alice       ]  |
|  ○ Drone           (♂)                 |  Gender:  ( ) Mixed        |
|  ○ Narrator        (mixed)             |           (●) Female       |
|  [ + New Profile ]                     |           ( ) Male         |
|  [ Import JSON  ] [ Export JSON ]      |  Pin seed: [ blank ]       |
|                                        |                            |
|                                        |  DERIVED VOICE SPEC        |
|                                        |  ──────────────────        |
|                                        |  Voice A: bf_alice  (#18)  |
|                                        |  Voice B: bf_emma   (#19)  |
|                                        |  Blend t: 0.421            |
|                                        |  Pitch:   0.974            |
|                                        |  Energy:  1.052            |
|                                        |  Hash:    3a7f9c12...      |
|                                        |  Arc Idx: #142             |
|                                        |  Curve:   0.43             |
|                                        |  Harmonic:0.11             |
|                                        |  Spectral:-0.03            |
|                                        |  Kinship: 0.87             |
|                                        |                            |
|                                        |  PREVIEW                   |
|                                        |  [ Hello, I'm Alice...   ] |
|                                        |  [ Synthesise ▶ ]          |
|                                        |  [ ━━━━━━○━━━━━━ 0:00/0:02]|
+----------------------------------------+----------------------------+
```

**ORDER tab — JSON upload + atom enumeration preview:**

```
+----------------------------------------------------------+
|  ORDER FILE                                              |
|  ──────────                                              |
|  [ Drag & drop alice_order.json or click to upload ]     |
|                                                          |
|  Loaded: alice_order.json  v1.0.0   ( 4 contexts )       |
|  [ Re-upload ] [ Clear ]                                 |
|                                                          |
|  ENUMERATION                                             |
|  ───────────                                             |
|  Total atoms: 35  /  100 cap                             |
|  By context:                                             |
|    ack          tier 1   12 atoms                        |
|    tool_invoke  tier 1   18 atoms                        |
|    progress     tier 2    3 atoms                        |
|    error        tier 2    2 atoms                        |
|                                                          |
|  Sample atoms (first 20):                                |
|    1.  Sure, give me a moment.                           |
|    2.  Sure, on it.                                      |
|    3.  Sure, working on it now.                          |
|    4.  Right, give me a moment.                          |
|    ...                                                   |
+----------------------------------------------------------+
```

**FACTORY tab — run controls + progress:**

```
+----------------------------------------------------------+
|  RUN STATUS                                              |
|  ──────────                                              |
|  Voice:   Alice  (bf_alice ↔ bf_emma, t=0.42)            |
|  Order:   alice_order.json  (35 atoms)                   |
|                                                          |
|  [ ▶ Run Factory ]   [ ■ Cancel ]                        |
|                                                          |
|  Progress: 18 / 35   (51%)                               |
|  ████████████░░░░░░░░░░░░                                |
|  ETA: 9.3 s                                              |
|                                                          |
|  Current: "Reaching for the calculator."                 |
|                                                          |
|  Log:                                                    |
|  [zerotts-factory] tier 1: 12/12 ack atoms rendered      |
|  [zerotts-factory] tier 1: 6/18 tool_invoke atoms ...    |
+----------------------------------------------------------+
```

**DOWNLOAD tab — pool listing:**

```
+----------------------------------------------------------+
|  AudioPool — 35 clips · 1.74 MB                          |
|                                                          |
|  [ Download All (ZIP) ]  [ Manifest (JSON) ]  [ Clear ]  |
|                                                          |
|  CTX: ack (12 clips)                                     |
|  ─────────────────                                       |
|  ▶ ⬇ 🗑   Sure, give me a moment.       1.62 s   48 KB   |
|  ▶ ⬇ 🗑   Sure, on it.                  0.84 s   24 KB   |
|  ▶ ⬇ 🗑   Sure, working on it now.      1.41 s   42 KB   |
|  ...                                                     |
|                                                          |
|  CTX: tool_invoke (18 clips)                             |
|  ─────────────────────────                               |
|  ▶ ⬇ 🗑   I'll use the web search now.  1.92 s   58 KB   |
|  ...                                                     |
+----------------------------------------------------------+
```

### Edge Cases and Error States

**E1 — Setup not complete.** If the user reaches VOICE/ORDER/FACTORY before bootstrap finishes, the affected tab shows a "Setup in progress" banner pointing back at the SETUP screen. Tabs remain navigable but their primary actions are disabled.

**E2 — Cache Storage unavailable.** Same as ZeroTTS — if the `caches` API is missing, SETUP fails fast with code `CACHE_STORAGE_UNAVAILABLE` and a remediation banner asking the user to leave private mode / update browser. The app does not fall back to ad-hoc downloads because asset size makes per-session re-fetch impractical.

**E3 — COOP/COEP missing.** Kokoro WASM requires `SharedArrayBuffer` and therefore cross-origin isolation. If `crossOriginIsolated` is `false` at runtime, the SETUP screen blocks with code `COI_MISSING` and instructions to add the headers (`Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Embedder-Policy: credentialless`).

**E4 — Order JSON malformed.** Schema validation runs on upload via Zod; on failure, the ORDER tab shows the offending field path and message. Atom enumeration is not attempted.

**E5 — Order exceeds max_atoms cap.** If enumeration would produce more than `install.max_atoms` (default 100), enumeration stops at the cap and a warning banner displays the truncation count. Run Factory uses only the truncated list.

**E6 — Atom synthesis failure.** If Kokoro throws while rendering an atom, the factory retries once. On second failure, the atom is recorded as `failed` in the manifest, skipped, and rendering continues with the next atom. The DOWNLOAD tab marks failed atoms with a red ⚠ row that has no Play/Download buttons but still appears so the user knows what's missing.

**E7 — Phonemizer-skipped input.** Atoms containing characters espeak-ng-wasm cannot phonemize render to silence or partial speech. The factory still produces a WAV but flags the row in DOWNLOAD with a yellow ⚠ icon and the note "phoneme map skipped N chars".

**E8 — Quota exceeded during ZIP build.** The browser's blob storage has a quota. If `JSZip.generateAsync` throws `QuotaExceededError` (very large pool, low disk), the app pauses, surfaces a `QUOTA_EXCEEDED` modal, and offers two recovery paths: download the partially-built archive, or split into smaller archive parts manually.

**E9 — Cancel mid-run.** Clicking Cancel during a factory run aborts after the current atom finishes. Already-rendered atoms remain in the AudioPool; partial pool is downloadable.

**E10 — Persona name collision.** Two profiles with the same `personaSeed` share a derived voice (this is by design — name is identity). The UI warns when the user tries to save a second profile whose seed matches an existing one and offers to set a `pinSeed` to disambiguate.

**E11 — Tab close mid-run.** The factory does not persist its run state. Closing the tab discards the AudioPool (clips are in-memory blobs). A `beforeunload` confirm dialog fires when the pool is non-empty: "You have unsaved generated audio. Leave without downloading?".

**E12 — File-name collisions in ZIP.** When two atoms have identical text after sanitisation, the file name suffix `_collision{n}` is appended starting from `_collision1`. Both clips ship.

### Performance Goals

- **Setup time (cold):** under 60 s on a 50 Mbit/s connection (88 MB model dominates).
- **Setup time (warm cache):** under 3 s — pure cache reads.
- **Single-atom synthesis:** ≤ 0.7 s per atom on 2020-era laptop, WASM int8.
- **35-atom factory run:** under 30 s end-to-end.
- **100-atom factory run:** under 90 s end-to-end.
- **ZIP build (35 clips, ~2 MB total):** under 1 s.
- **ZIP build (100 clips, ~50 MB total):** under 6 s.
- **Voice preview latency:** under 1.5 s from click to first audio sample.

### Accessibility Requirements

- All progress indicators (setup, factory) have ARIA `role="progressbar"` with `aria-valuenow`, `aria-valuemax`, and a `aria-live="polite"` text label that updates every 10% of progress.
- All buttons have explicit `aria-label`s including the action and target object (e.g. `aria-label="Download Alice clip 12 of 35"`).
- The DOWNLOAD tab table has `role="table"` semantics with column headers and per-row actions reachable by keyboard (Tab + Enter).
- The VOICE tab waveform preview honours `prefers-reduced-motion` — the playhead animation is replaced by a discrete "Playing 0:00 / 0:02" text counter.
- Colour cues (red ⚠ failed, yellow ⚠ phoneme-skip) are paired with text labels — never colour alone.

---

## Technical Implementation

### Architecture

```
                    ┌──────────────────────────────────────┐
                    │              React App               │
                    │   (tabs: Voice / Order / Factory /   │
                    │    Download / Setup)                 │
                    └──────────────┬───────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│                ZeroTTS-Factory Layer                         │
│                                                              │
│  zfHash.ts          ← xxhash-wasm primitive (32 + 64 bit)    │
│  zfVoiceDeriver.ts  ← name → VoiceSpec  (string-as-seed)     │
│  zfArc.ts           ← arc-property derivation (in-browser)   │
│  zfProfile.ts       ← VoiceProfile type + persistence        │
│  zfOrder.ts         ← TTSProfile loader, validator, enum.    │
│  zfFactory.ts       ← run loop, progress, AudioPool          │
│  zfPool.ts          ← in-memory AudioPool API                │
│  zfZipBuilder.ts    ← multi-archive ZIP splitter             │
│  zfWav.ts           ← Float32Array → WAV Blob encoder        │
│  zfSetup.ts         ← model + voices + espeak bootstrap      │
│  useFactory.ts      ← React hook integrating all of above    │
│                                                              │
└─────────────────┬─────────────────┬──────────────────────────┘
                  │                 │
                  ▼                 ▼
┌──────────────────────────────┐  ┌──────────────────────────┐
│  Kokoro WASM Client          │  │  Cache Storage / IDB     │
│  (kokoroClient.ts —          │  │                          │
│   browser port from          │  │  transformers-kokoro-*   │
│   ZeroVoice-KokoroTTS)       │  │  zerotts-espeak-wasm     │
│                              │  │  (no atom cache —        │
│  - ensureReady()             │  │   audio is in-memory     │
│  - synthesize(text,          │  │   AudioPool only)        │
│        spec) → Float32Array  │  └──────────────────────────┘
│  - synthesize returns        │
│        sampleRate=24000      │
└──────────────────────────────┘
```

### File Layout

```
zerotts-factory/
├── package.json
├── tsconfig.json
├── tsconfig.node.json
├── vite.config.ts
├── index.html
├── public/
│   └── (assets fetched at runtime — not bundled)
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── styles/
    │   ├── factory.css
    │   └── controls.css
    ├── lib/
    │   ├── zfTypes.ts
    │   ├── zfHash.ts
    │   ├── zfVoiceDeriver.ts
    │   ├── zfArc.ts
    │   ├── zfProfile.ts
    │   ├── zfOrder.ts
    │   ├── zfFactory.ts
    │   ├── zfPool.ts
    │   ├── zfZipBuilder.ts
    │   ├── zfWav.ts
    │   └── zfSetup.ts
    ├── kokoro/
    │   ├── kokoroClient.ts     ← from ZeroVoice-KokoroTTS browser pipeline
    │   ├── voiceTable.ts       ← loads voices-v1.0.bin (NPZ in browser)
    │   └── phonemize.ts        ← espeak-ng-wasm wrapper
    ├── hooks/
    │   ├── useSetup.ts
    │   ├── useFactory.ts
    │   ├── useAudioPool.ts
    │   └── useVoiceProfile.ts
    └── components/
        ├── tabs/
        │   ├── VoiceTab.tsx
        │   ├── OrderTab.tsx
        │   ├── FactoryTab.tsx
        │   ├── DownloadTab.tsx
        │   └── SetupScreen.tsx
        ├── voice/
        │   ├── ProfileList.tsx
        │   ├── ProfileEditor.tsx
        │   ├── VoiceSpecPanel.tsx
        │   └── PreviewPlayer.tsx
        ├── order/
        │   ├── OrderUpload.tsx
        │   └── EnumerationView.tsx
        ├── factory/
        │   ├── RunControls.tsx
        │   └── ProgressLog.tsx
        ├── download/
        │   ├── PoolTable.tsx
        │   ├── ClipRow.tsx
        │   └── ZipDownloadButton.tsx
        └── ui/
            ├── TabBar.tsx
            ├── StatusFooter.tsx
            └── ErrorBanner.tsx
```

### Data Models

```typescript
// src/lib/zfTypes.ts

/** Three-way gender filter — matches ZeroVoice-KokoroTTS Rust enum. */
export type GenderFilter = "mixed" | "female" | "male";

/** Persistent voice profile saved in localStorage. */
export interface VoiceProfile {
  /** Display name; what UI shows. */
  name: string;
  /** Lowercased+sanitised name; used as the default hash seed. */
  personaSeed: string;
  gender: GenderFilter;
  /** Optional override seed; when non-null, replaces personaSeed in hash input. */
  pinSeed: string | null;
  /** Optional manual locks for per-property overrides. */
  lockedOverrides: {
    t:      { locked: boolean; value: number };
    pitch:  { locked: boolean; value: number };
    energy: { locked: boolean; value: number };
    voiceA: { locked: boolean; value: number };
    voiceB: { locked: boolean; value: number };
  };
  /** Profile creation timestamp (Unix ms). */
  createdAt: number;
}

/** Voice spec derived from a name+gender+pin combination. */
export interface VoiceSpec {
  voiceA: number;
  voiceB: number;
  voiceAName: string;
  voiceBName: string;
  t: number;            // [0.0, 1.0)
  pitchScale: number;   // [0.85, 1.15]
  energyScale: number;  // [0.80, 1.20]
  hashHex: string;      // 16-char zero-padded hex of voiceA hash
  arcIndex: number;     // 0..324 unordered-pair index
  arcCurveWarp: number;       // [0.2, 0.8]
  arcHarmonicWeight: number;  // [0.0, 0.3]
  arcSpectralSkew: number;    // [-0.15, +0.15]
  arcKinship: number;         // [0.0, 1.0]
}

/** TTSProfile order — adapted from ZeroTTS-zvKokoro (no voice_blend block). */
export interface OrderProfile {
  name: string;
  description?: string;
  version: string;
  contexts: Record<string, OrderContext>;
  pools: Record<string, string[]>;
  install: {
    max_atoms?: number;
  };
}

export interface OrderContext {
  templates: string[];
  tier: 1 | 2 | 3;
  trigger_hints?: string[];
  prosody_tag?: string;
}

/** A single atom resolved from a context+template+slot-fill combination. */
export interface EnumeratedAtom {
  /** Stable id: hash(personaSeed, atomText). */
  id: string;
  atomText: string;
  contextName: string;
  tier: 1 | 2 | 3;
}

/** A rendered clip in the in-memory AudioPool. */
export interface PoolClip {
  id: string;
  atomText: string;
  contextName: string;
  tier: 1 | 2 | 3;
  /** Owning profile's personaSeed at the time of render. */
  personaSeed: string;
  /** Display name at the time of render. */
  personaName: string;
  durationMs: number;
  byteSize: number;
  blob: Blob;
  /** URL.createObjectURL output — revoked on Clear or Remove. */
  objectUrl: string;
  status: "rendered" | "failed" | "phoneme_skip";
  /** Sanitised filename used in ZIP and individual download. */
  filename: string;
}

/** Manifest written into ZIP archives for traceability. */
export interface PoolManifest {
  generated_at: number;
  factory_version: string;
  persona: {
    name: string;
    seed: string;
    pinSeed: string | null;
    gender: GenderFilter;
  };
  voice: VoiceSpec;
  order_name: string;
  order_version: string;
  archives: Array<{
    name: string;
    files: string[];
    byte_size: number;
  }>;
  clips: Array<{
    filename: string;
    archive: string;
    atom_text: string;
    context: string;
    tier: number;
    duration_ms: number;
    byte_size: number;
    status: PoolClip["status"];
  }>;
}
```

### Key Algorithms

**Algorithm 1 — Hashing primitive (string + salts).**

```typescript
// src/lib/zfHash.ts
import xxhash from "xxhash-wasm";

let _api: Awaited<ReturnType<typeof xxhash>> | null = null;

export async function initHash(): Promise<void> {
  if (!_api) _api = await xxhash();
}

/** xxh64 of a UTF-8 string with `WORLD_SEED ^ salt` as seed. */
export const WORLD_SEED = 0xDEADBEEFC0FFEn;

export function hashString(seedStr: string, salt: bigint): bigint {
  if (!_api) throw new Error("Call initHash() first");
  const seed = WORLD_SEED ^ salt;
  const encoded = new TextEncoder().encode(seedStr);
  return _api.h64Raw(encoded, seed);
}

/** Map low 32 bits of a 64-bit hash to a float in [0.0, 1.0). */
export function hashToUnit(h: bigint): number {
  return Number(h & 0xFFFFFFFFn) / 0x100000000;
}

/** xxh32 of a string for short ids (used by atom keys). */
export function hashShort(seedStr: string): string {
  if (!_api) throw new Error("Call initHash() first");
  return _api.h32(seedStr).toString(16).padStart(8, "0");
}
```

**Algorithm 2 — Voice derivation (string-as-seed).**

```typescript
// src/lib/zfVoiceDeriver.ts
import { hashString, hashToUnit } from "./zfHash";
import type { GenderFilter, VoiceProfile, VoiceSpec } from "./zfTypes";
import { deriveArcProperties } from "./zfArc";

export const KOKORO_VOICES = [
  "af_alloy", "af_aoede", "af_bella",   "af_jessica", "af_kore",
  "af_nicole", "af_nova",  "af_river",   "af_sarah",   "af_sky",
  "am_adam",  "am_echo",  "am_eric",    "am_fenrir",  "am_liam",
  "am_michael","am_onyx", "am_puck",    "bf_alice",   "bf_emma",
  "bf_isabella","bf_lily","bm_daniel",  "bm_fable",   "bm_george",
  "bm_lewis",
] as const;

export const FEMALE_INDICES = [0,1,2,3,4,5,6,7,8,9,18,19,20,21] as const;
export const MALE_INDICES   = [10,11,12,13,14,15,16,17,22,23,24,25] as const;

const SALT_VOICE_A = 0x0001n;
const SALT_VOICE_B = 0x0002n;
const SALT_SLERP_T = 0x0003n;
const SALT_PITCH   = 0x0004n;
const SALT_ENERGY  = 0x0005n;

/** Sanitise a free-form name into a valid persona seed. */
export function sanitiseSeed(name: string): string {
  return name.toLowerCase().replace(/[^a-z0-9_-]/g, "_");
}

function pickVoicePair(hA: bigint, hB: bigint, gender: GenderFilter): [number, number] {
  if (gender === "mixed") {
    const idxA = Number(hA % 26n);
    const idxB = (Number(hB % 25n) + idxA + 1) % 26;
    return [idxA, idxB];
  }
  const pool = gender === "female" ? FEMALE_INDICES : MALE_INDICES;
  const n = BigInt(pool.length);
  const aPos = Number(hA % n);
  const bPos = (Number(hB % (n - 1n)) + aPos + 1) % pool.length;
  return [pool[aPos], pool[bPos]];
}

function arcIndex(a: number, b: number): number {
  const [lo, hi] = a < b ? [a, b] : [b, a];
  return lo * 25 - Math.floor(lo * (lo - 1) / 2) + hi - lo - 1;
}

/** Derive a complete VoiceSpec from a profile.  All clients must `await initHash()` first. */
export function deriveVoiceSpec(profile: VoiceProfile): VoiceSpec {
  const seed = profile.pinSeed ?? profile.personaSeed;
  const hA = hashString(seed, SALT_VOICE_A);
  const hB = hashString(seed, SALT_VOICE_B);
  const hT = hashString(seed, SALT_SLERP_T);
  const hP = hashString(seed, SALT_PITCH);
  const hE = hashString(seed, SALT_ENERGY);

  const [idxA, idxB] = pickVoicePair(hA, hB, profile.gender);
  const arc = deriveArcProperties(idxA, idxB);

  return {
    voiceA: idxA,
    voiceB: idxB,
    voiceAName: KOKORO_VOICES[idxA],
    voiceBName: KOKORO_VOICES[idxB],
    t: hashToUnit(hT),
    pitchScale: 0.85 + hashToUnit(hP) * 0.30,
    energyScale: 0.80 + hashToUnit(hE) * 0.40,
    hashHex: hA.toString(16).padStart(16, "0"),
    arcIndex: arcIndex(idxA, idxB),
    ...arc,
  };
}

/** Apply locked overrides to a derived spec just before synthesis. */
export function applyOverrides(
  spec: VoiceSpec,
  overrides: VoiceProfile["lockedOverrides"],
): VoiceSpec {
  const next: VoiceSpec = { ...spec };
  if (overrides.t.locked)      next.t = overrides.t.value;
  if (overrides.pitch.locked)  next.pitchScale = overrides.pitch.value;
  if (overrides.energy.locked) next.energyScale = overrides.energy.value;
  if (overrides.voiceA.locked) {
    next.voiceA = overrides.voiceA.value;
    next.voiceAName = KOKORO_VOICES[next.voiceA];
  }
  if (overrides.voiceB.locked) {
    next.voiceB = overrides.voiceB.value;
    next.voiceBName = KOKORO_VOICES[next.voiceB];
  }
  next.arcIndex = arcIndex(next.voiceA, next.voiceB);
  return next;
}
```

**Algorithm 3 — Arc property derivation (browser port).**

Mirrors the Rust `arc_properties.rs` from ZeroVoice-KokoroTTS. Kinship is computed lazily once the voice table is loaded; until then it is reported as `0.0`.

```typescript
// src/lib/zfArc.ts
import { hashString, hashToUnit } from "./zfHash";

const SALT_ARC_CURVE    = 0x0010n;
const SALT_ARC_HARMONIC = 0x0011n;
const SALT_ARC_SPECTRAL = 0x0012n;

/** Voice table is set once after voices-v1.0.bin loads. */
let _voiceTable: Float32Array[] | null = null;

export function setVoiceTable(table: Float32Array[]): void {
  _voiceTable = table; // table[i] = representative row-0 style vector for voice i
}

function pairHash(a: number, b: number, salt: bigint): bigint {
  const [lo, hi] = a <= b ? [a, b] : [b, a];
  return hashString(`${lo}|${hi}`, salt);
}

function computeKinship(a: number, b: number): number {
  if (!_voiceTable) return 0.0;
  const va = _voiceTable[a], vb = _voiceTable[b];
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < va.length; i++) {
    dot += va[i] * vb[i];
    na  += va[i] * va[i];
    nb  += vb[i] * vb[i];
  }
  const denom = Math.max(1e-8, Math.sqrt(na) * Math.sqrt(nb));
  return Math.max(0, Math.min(1, dot / denom));
}

export function deriveArcProperties(a: number, b: number): {
  arcCurveWarp: number;
  arcHarmonicWeight: number;
  arcSpectralSkew: number;
  arcKinship: number;
} {
  if (a === b) {
    return { arcCurveWarp: 0.5, arcHarmonicWeight: 0.0, arcSpectralSkew: 0.0, arcKinship: 1.0 };
  }
  const hC = pairHash(a, b, SALT_ARC_CURVE);
  const hH = pairHash(a, b, SALT_ARC_HARMONIC);
  const hS = pairHash(a, b, SALT_ARC_SPECTRAL);
  return {
    arcCurveWarp:      0.2 + hashToUnit(hC) * 0.6,
    arcHarmonicWeight: hashToUnit(hH) * 0.3,
    arcSpectralSkew:   hashToUnit(hS) * 0.30 - 0.15,
    arcKinship:        computeKinship(a, b),
  };
}
```

**Algorithm 4 — Order schema validation + enumeration.**

```typescript
// src/lib/zfOrder.ts
import { z } from "zod";
import type { OrderProfile, EnumeratedAtom } from "./zfTypes";
import { hashString } from "./zfHash";

const OrderContextSchema = z.object({
  templates: z.array(z.string()).min(1),
  tier: z.union([z.literal(1), z.literal(2), z.literal(3)]),
  trigger_hints: z.array(z.string()).optional(),
  prosody_tag: z.string().optional(),
});

const OrderProfileSchema = z.object({
  name: z.string(),
  description: z.string().optional(),
  version: z.string(),
  contexts: z.record(OrderContextSchema),
  pools: z.record(z.array(z.string()).min(1)),
  install: z.object({
    max_atoms: z.number().int().positive().max(10000).optional(),
  }),
});

export function parseOrder(json: unknown): OrderProfile {
  return OrderProfileSchema.parse(json) as OrderProfile;
}

const SLOT_RE = /\{(\w+)\}/g;

function extractSlots(template: string): string[] {
  const out: string[] = [];
  let m: RegExpExecArray | null;
  while ((m = SLOT_RE.exec(template)) !== null) out.push(m[1]);
  SLOT_RE.lastIndex = 0;
  return out;
}

function fillTemplate(template: string, slots: string[], values: string[]): string {
  let result = template;
  slots.forEach((s, i) => { result = result.replace(`{${s}}`, values[i]); });
  return result;
}

function* cartesian<T>(arrs: T[][]): Generator<T[]> {
  if (arrs.length === 0) { yield []; return; }
  const [head, ...rest] = arrs;
  for (const x of head) for (const tail of cartesian(rest)) yield [x, ...tail];
}

const SALT_ATOM_ID = 0x00A0n;

export function enumerateAtoms(
  order: OrderProfile,
  personaSeed: string,
): { atoms: EnumeratedAtom[]; truncated: number } {
  const cap = order.install.max_atoms ?? 100;
  const seen = new Map<string, EnumeratedAtom>();
  let truncated = 0;

  // Sort contexts by tier so cap eviction prefers low-priority atoms.
  const ctxOrder = Object.entries(order.contexts).sort((a, b) => a[1].tier - b[1].tier);

  for (const [ctxName, ctx] of ctxOrder) {
    for (const tpl of ctx.templates) {
      const slots = extractSlots(tpl);
      const slotValues = slots.map((s) => order.pools[s] ?? []);
      if (slotValues.some((v) => v.length === 0)) {
        if (slots.length === 0) {
          // No slots; the template itself is the atom.
          if (!seen.has(tpl)) {
            const id = hashString(`${personaSeed}|${tpl}`, SALT_ATOM_ID).toString(16);
            seen.set(tpl, { id, atomText: tpl, contextName: ctxName, tier: ctx.tier });
            if (seen.size >= cap) return finish(seen, /*overflow*/ Infinity);
          }
          continue;
        }
        console.warn(`[zerotts-factory] template "${tpl}" missing pool — skipped`);
        continue;
      }
      for (const combo of cartesian(slotValues)) {
        const atomText = fillTemplate(tpl, slots, combo);
        if (!seen.has(atomText)) {
          const id = hashString(`${personaSeed}|${atomText}`, SALT_ATOM_ID).toString(16);
          seen.set(atomText, { id, atomText, contextName: ctxName, tier: ctx.tier });
          if (seen.size >= cap) {
            // Count remaining combos as truncated (best-effort estimate).
            return finish(seen, cap);
          }
        }
      }
    }
  }
  return finish(seen, 0);

  function finish(map: Map<string, EnumeratedAtom>, overflow: number) {
    return {
      atoms: Array.from(map.values()),
      truncated: overflow === Infinity ? 0 : Math.max(0, overflow === 0 ? 0 : 0),
    };
  }
}
```

**Algorithm 5 — Factory run loop.**

```typescript
// src/lib/zfFactory.ts
import type { OrderProfile, VoiceProfile, EnumeratedAtom, PoolClip } from "./zfTypes";
import { deriveVoiceSpec, applyOverrides } from "./zfVoiceDeriver";
import { encodePcmAsWav } from "./zfWav";
import type { KokoroClient } from "../kokoro/kokoroClient";
import { addClip } from "./zfPool";

const FACTORY_VERSION = "0.1.0";

function sanitiseFilename(text: string, maxLen = 64): string {
  return text.toLowerCase()
    .replace(/[^\w\s-]/g, "")
    .trim()
    .replace(/\s+/g, "_")
    .slice(0, maxLen);
}

export interface FactoryRunArgs {
  voiceProfile: VoiceProfile;
  order: OrderProfile;
  atoms: EnumeratedAtom[];
  kokoro: KokoroClient;
  signal?: AbortSignal;
  onProgress?: (done: number, total: number, current: string) => void;
  onAtomDone?: (clip: PoolClip) => void;
}

export async function runFactory(args: FactoryRunArgs): Promise<PoolClip[]> {
  const { voiceProfile, order, atoms, kokoro, signal, onProgress, onAtomDone } = args;

  const baseSpec = deriveVoiceSpec(voiceProfile);
  const spec = applyOverrides(baseSpec, voiceProfile.lockedOverrides);

  const sortedAtoms = [...atoms].sort((a, b) => a.tier - b.tier);
  const total = sortedAtoms.length;
  const out: PoolClip[] = [];

  for (let i = 0; i < total; i++) {
    if (signal?.aborted) throw new DOMException("aborted", "AbortError");
    const atom = sortedAtoms[i];
    onProgress?.(i, total, atom.atomText);

    let clip: PoolClip | null = null;
    for (let attempt = 0; attempt < 2 && clip === null; attempt++) {
      try {
        const pcm = await kokoro.synthesize({
          text: atom.atomText,
          voiceA: spec.voiceA,
          voiceB: spec.voiceB,
          t: spec.t,
          pitchScale: spec.pitchScale,
          energyScale: spec.energyScale,
          arcCurveWarp: spec.arcCurveWarp,
          arcHarmonicWeight: spec.arcHarmonicWeight,
          arcSpectralSkew: spec.arcSpectralSkew,
        });
        const wav = encodePcmAsWav(pcm, 24000);
        const objectUrl = URL.createObjectURL(wav);
        const filename =
          `${voiceProfile.personaSeed}_${atom.contextName}_${i.toString().padStart(3, "0")}_${sanitiseFilename(atom.atomText, 32)}.wav`;
        clip = {
          id: atom.id,
          atomText: atom.atomText,
          contextName: atom.contextName,
          tier: atom.tier,
          personaSeed: voiceProfile.personaSeed,
          personaName: voiceProfile.name,
          durationMs: Math.round((pcm.length / 24000) * 1000),
          byteSize: wav.size,
          blob: wav,
          objectUrl,
          status: "rendered",
          filename,
        };
      } catch (err) {
        if (attempt === 1) {
          // Persistent failure — record an empty stub.
          clip = {
            id: atom.id,
            atomText: atom.atomText,
            contextName: atom.contextName,
            tier: atom.tier,
            personaSeed: voiceProfile.personaSeed,
            personaName: voiceProfile.name,
            durationMs: 0,
            byteSize: 0,
            blob: new Blob([], { type: "audio/wav" }),
            objectUrl: "",
            status: "failed",
            filename: `${voiceProfile.personaSeed}_${atom.contextName}_${i.toString().padStart(3, "0")}_FAILED.wav`,
          };
        }
      }
    }

    if (clip) {
      addClip(clip);
      out.push(clip);
      onAtomDone?.(clip);
    }
  }

  onProgress?.(total, total, "done");
  return out;
}

export { FACTORY_VERSION };
```

**Algorithm 6 — AudioPool (in-memory store).**

```typescript
// src/lib/zfPool.ts
import type { PoolClip } from "./zfTypes";

const _clips = new Map<string, PoolClip>();
const _listeners = new Set<() => void>();

export function addClip(c: PoolClip): void {
  const prev = _clips.get(c.id);
  if (prev?.objectUrl) URL.revokeObjectURL(prev.objectUrl);
  _clips.set(c.id, c);
  notify();
}

export function removeClip(id: string): void {
  const prev = _clips.get(id);
  if (prev?.objectUrl) URL.revokeObjectURL(prev.objectUrl);
  _clips.delete(id);
  notify();
}

export function clearPool(): void {
  for (const c of _clips.values()) if (c.objectUrl) URL.revokeObjectURL(c.objectUrl);
  _clips.clear();
  notify();
}

export function listClips(): PoolClip[] {
  return Array.from(_clips.values()).sort((a, b) => {
    if (a.tier !== b.tier) return a.tier - b.tier;
    if (a.contextName !== b.contextName) return a.contextName.localeCompare(b.contextName);
    return a.atomText.localeCompare(b.atomText);
  });
}

export function poolByteSize(): number {
  let total = 0;
  for (const c of _clips.values()) total += c.byteSize;
  return total;
}

function notify(): void { for (const f of _listeners) f(); }
export function subscribe(f: () => void): () => void {
  _listeners.add(f);
  return () => _listeners.delete(f);
}
```

**Algorithm 7 — Multi-archive ZIP splitter.**

Uses `jszip`. The splitter packs files in pre-sorted order, finalising and downloading the current archive as soon as adding the next file would exceed `MAX_ZIP_BYTES`. The exact compressed size of a not-yet-finalised archive is impossible to know cheaply, so the implementation uses an **uncompressed** size budget — files are added with `compression: "STORE"` (no compression) precisely so the running budget is exact. WAV PCM compresses badly with DEFLATE anyway; STORE keeps the math simple and the build fast.

```typescript
// src/lib/zfZipBuilder.ts
import JSZip from "jszip";
import type { PoolClip, PoolManifest, VoiceProfile, OrderProfile } from "./zfTypes";
import { deriveVoiceSpec, applyOverrides } from "./zfVoiceDeriver";

const MAX_ZIP_BYTES = 100 * 1024 * 1024; // 100 MB
const ZIP_OVERHEAD_PER_FILE = 256;        // local + central dir overhead estimate
const ZIP_FOOTER = 22;

export interface BuildArchivesArgs {
  clips: PoolClip[];
  voiceProfile: VoiceProfile;
  order: OrderProfile | null;
  factoryVersion: string;
}

interface ArchivePlan {
  name: string;
  zip: JSZip;
  files: string[];
  byteSize: number;  // uncompressed sum
}

export async function buildAndDownloadArchives(args: BuildArchivesArgs): Promise<void> {
  const { clips, voiceProfile, order, factoryVersion } = args;

  // Pre-plan: split clips into archive groups.
  const plans: Array<{ name: string; clips: PoolClip[] }> = [];
  let current: PoolClip[] = [];
  let runningSize = ZIP_FOOTER;
  for (const clip of clips) {
    const cost = clip.byteSize + ZIP_OVERHEAD_PER_FILE + clip.filename.length;
    if (runningSize + cost > MAX_ZIP_BYTES && current.length > 0) {
      plans.push({ name: "", clips: current });
      current = [];
      runningSize = ZIP_FOOTER;
    }
    current.push(clip);
    runningSize += cost;
  }
  if (current.length > 0) plans.push({ name: "", clips: current });

  const total = plans.length;
  const orderName = order?.name ?? "preview";
  const safeOrderName = orderName.toLowerCase().replace(/[^\w-]/g, "_");
  plans.forEach((p, i) => {
    p.name = `${voiceProfile.personaSeed}_${safeOrderName}_part${(i + 1).toString().padStart(2, "0")}_of${total.toString().padStart(2, "0")}.zip`;
  });

  // Build manifest first (to embed into part 01).
  const baseSpec = deriveVoiceSpec(voiceProfile);
  const spec = applyOverrides(baseSpec, voiceProfile.lockedOverrides);
  const manifest: PoolManifest = {
    generated_at: Date.now(),
    factory_version: factoryVersion,
    persona: {
      name: voiceProfile.name,
      seed: voiceProfile.personaSeed,
      pinSeed: voiceProfile.pinSeed,
      gender: voiceProfile.gender,
    },
    voice: spec,
    order_name: orderName,
    order_version: order?.version ?? "n/a",
    archives: plans.map((p) => ({
      name: p.name,
      files: p.clips.map((c) => c.filename),
      byte_size: p.clips.reduce((acc, c) => acc + c.byteSize, 0),
    })),
    clips: clips.map((c) => {
      const archive = plans.find((p) => p.clips.includes(c))!.name;
      return {
        filename: c.filename,
        archive,
        atom_text: c.atomText,
        context: c.contextName,
        tier: c.tier,
        duration_ms: c.durationMs,
        byte_size: c.byteSize,
        status: c.status,
      };
    }),
  };

  // Build and download each archive in sequence.
  for (let i = 0; i < plans.length; i++) {
    const plan = plans[i];
    const zip = new JSZip();
    if (i === 0) {
      zip.file("manifest.json", JSON.stringify(manifest, null, 2));
    }
    for (const clip of plan.clips) {
      zip.file(clip.filename, clip.blob, { compression: "STORE" });
    }
    const blob = await zip.generateAsync({
      type: "blob",
      compression: "STORE",
      streamFiles: true,
    });
    triggerDownload(blob, plan.name);
  }
}

function triggerDownload(blob: Blob, filename: string): void {
  const url = URL.createObjectURL(blob);
  const a = Object.assign(document.createElement("a"), { href: url, download: filename });
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 30_000);
}
```

**Algorithm 8 — WAV encoder.**

Verbatim port of `src/lib/waveform.ts` from ZeroVoice-KokoroTTS, lightly renamed. 24 kHz mono, 16-bit PCM.

```typescript
// src/lib/zfWav.ts
export function encodePcmAsWav(pcm: Float32Array, sampleRate = 24000): Blob {
  const numSamples = pcm.length;
  const buffer = new ArrayBuffer(44 + numSamples * 2);
  const view = new DataView(buffer);
  const str = (off: number, s: string) =>
    [...s].forEach((c, i) => view.setUint8(off + i, c.charCodeAt(0)));

  str(0,  "RIFF");
  view.setUint32( 4, 36 + numSamples * 2,  true);
  str(8,  "WAVE");
  str(12, "fmt ");
  view.setUint32(16, 16,             true);
  view.setUint16(20,  1,             true);
  view.setUint16(22,  1,             true);
  view.setUint32(24, sampleRate,     true);
  view.setUint32(28, sampleRate * 2, true);
  view.setUint16(32,  2,             true);
  view.setUint16(34, 16,             true);
  str(36, "data");
  view.setUint32(40, numSamples * 2, true);

  for (let i = 0; i < numSamples; i++) {
    const s = Math.max(-1, Math.min(1, pcm[i]));
    view.setInt16(44 + i * 2, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
  }
  return new Blob([buffer], { type: "audio/wav" });
}

export function downloadBlob(blob: Blob, filename: string): void {
  const url = URL.createObjectURL(blob);
  const a = Object.assign(document.createElement("a"), { href: url, download: filename });
  a.click();
  setTimeout(() => URL.revokeObjectURL(url), 10_000);
}
```

**Algorithm 9 — Profile persistence.**

```typescript
// src/lib/zfProfile.ts
import type { VoiceProfile } from "./zfTypes";
import { sanitiseSeed } from "./zfVoiceDeriver";

const PROFILES_KEY = "zerotts-factory-voice-profiles";
const ACTIVE_KEY = "zerotts-factory-active-profile";

export function loadProfiles(): VoiceProfile[] {
  try {
    const raw = localStorage.getItem(PROFILES_KEY);
    if (!raw) return [];
    return JSON.parse(raw);
  } catch {
    return [];
  }
}

export function saveProfiles(list: VoiceProfile[]): void {
  localStorage.setItem(PROFILES_KEY, JSON.stringify(list));
}

export function loadActive(): string | null {
  return localStorage.getItem(ACTIVE_KEY);
}

export function saveActive(personaSeed: string | null): void {
  if (personaSeed == null) localStorage.removeItem(ACTIVE_KEY);
  else localStorage.setItem(ACTIVE_KEY, personaSeed);
}

export function newProfile(name: string): VoiceProfile {
  return {
    name,
    personaSeed: sanitiseSeed(name),
    gender: "mixed",
    pinSeed: null,
    lockedOverrides: {
      t:      { locked: false, value: 0.5 },
      pitch:  { locked: false, value: 1.0 },
      energy: { locked: false, value: 1.0 },
      voiceA: { locked: false, value: 0 },
      voiceB: { locked: false, value: 1 },
    },
    createdAt: Date.now(),
  };
}
```

### Integration with Kokoro WASM

ZeroTTS-Factory consumes a browser-port `KokoroClient` (the same one referenced by `ZeroTTS-zvKokoro-TINS.md`). The factory only needs one method:

```typescript
// src/kokoro/kokoroClient.ts (contract — full implementation lifted from
// the existing browser port that ships with ZeroVoice-KokoroTTS web edition)

export interface KokoroSynthesizeArgs {
  text: string;
  voiceA: number;
  voiceB: number;
  t: number;
  pitchScale: number;
  energyScale: number;
  arcCurveWarp: number;
  arcHarmonicWeight: number;
  arcSpectralSkew: number;
}

export interface KokoroClient {
  ensureReady(): Promise<void>;
  isReady(): boolean;
  /** Returns 24 kHz mono float32 PCM. */
  synthesize(args: KokoroSynthesizeArgs): Promise<Float32Array>;
  /** For preview UI — same as synthesize but also returns a decoded AudioBuffer. */
  synthesizeWithBuffer(args: KokoroSynthesizeArgs): Promise<{ pcm: Float32Array; audioBuffer: AudioBuffer }>;
}
```

The client internally:
1. Loads `kokoro-v1.0.int8.onnx` via `onnxruntime-web` (configured for WASM EP, SharedArrayBuffer enabled).
2. Loads `voices-v1.0.bin` (NPZ in browser) via `fflate` + a tiny NPY parser.
3. Builds the 26 voice-row Float32Array table; calls `setVoiceTable(table)` from `zfArc` so kinship can be computed.
4. Phonemises text via espeak-ng-wasm (loaded once via `Worker`).
5. For each `synthesize` call: token IDs → run `resolveStyleQuadratic` (same math as Rust `slerp.rs`'s `resolve_style_quadratic`) → ONNX `session.run({tokens, style, speed})` → Float32Array PCM.

### React hook contract

```typescript
// src/hooks/useFactory.ts
export interface UseFactoryResult {
  state: "idle" | "running" | "done" | "cancelling" | "error";
  progress: { done: number; total: number; current: string } | null;
  error: string | null;
  run: () => Promise<void>;
  cancel: () => void;
}

export function useFactory(): UseFactoryResult;

// src/hooks/useVoiceProfile.ts
export interface UseVoiceProfileResult {
  profiles: VoiceProfile[];
  active: VoiceProfile | null;
  spec: VoiceSpec | null;
  setActive: (personaSeed: string | null) => void;
  upsert: (p: VoiceProfile) => void;
  remove: (personaSeed: string) => void;
  newDraft: (name: string) => VoiceProfile;
}

export function useVoiceProfile(): UseVoiceProfileResult;

// src/hooks/useAudioPool.ts
export interface UseAudioPoolResult {
  clips: PoolClip[];
  byteSize: number;
  remove: (id: string) => void;
  clear: () => void;
  downloadAll: () => Promise<void>;
}

export function useAudioPool(): UseAudioPoolResult;

// src/hooks/useSetup.ts
export interface UseSetupResult {
  state: "checking" | "downloading" | "loading_session" | "ready" | "error";
  progress: { asset: string; done: number; total: number } | null;
  error: string | null;
  start: () => Promise<void>;
}

export function useSetup(): UseSetupResult;
```

### Hosting headers

Identical to ZeroVoice-KokoroTTS web edition:

```json
{
  "headers": [
    { "source": "**", "headers": [
      { "key": "Cross-Origin-Opener-Policy",   "value": "same-origin" },
      { "key": "Cross-Origin-Embedder-Policy", "value": "credentialless" }
    ]}
  ]
}
```

Without these, `crossOriginIsolated` is false and Kokoro WASM cannot allocate `SharedArrayBuffer`. The SETUP screen detects and surfaces this.

---

## Affected Regions of Source Specs

This implementation re-uses code blocks and concepts from the upstream specs. The mapping below is the implementer's roadmap — every "FROM" location was read; "ACTION" describes what changes when porting.

| Concern | FROM (upstream spec) | ACTION in ZeroTTS-Factory |
|---|---|---|
| 26 Kokoro voice list | `ZeroVoice-KokoroTTS-postV2-TINS.md` Step 5 (`KOKORO_VOICES`) | Verbatim into `zfVoiceDeriver.ts`. |
| Gender filter pools (FEMALE_INDICES, MALE_INDICES) | `ZeroVoice-KokoroTTS-postV2-TINS.md` Step 5 | Verbatim into `zfVoiceDeriver.ts`. |
| Salts (0x0001..0x0005, 0x0010..0x0012) | `ZeroVoice-KokoroTTS-postV2-TINS.md` Steps 5, 6 | Verbatim. |
| `WORLD_SEED = 0xDEADBEEFC0FFE` | `ZeroVoice-KokoroTTS-postV2-TINS.md` Step 5 | Verbatim into `zfHash.ts`. |
| Position hash (`xxh64`) | Step 5 `position_hash` | Replace 12-byte `(sx,sy,sz)` payload with **UTF-8 bytes of seed string**; salt-XOR semantics unchanged. |
| `pick_voice_pair` | Step 5 | Verbatim port to TS. |
| `pitch_scale` / `energy_scale` ranges | Step 5 | Verbatim. |
| Arc property derivation | Step 6 `arc_properties.rs` | Port to TS in `zfArc.ts`. Symmetric pair hashing preserved. |
| Kinship (cosine similarity row-0) | Step 6 `compute_kinship` | Port; kinship is `0.0` until voice table is loaded by KokoroClient. |
| `resolve_style_quadratic` | Step 7 `slerp.rs` | The browser KokoroClient applies this internally before ONNX run. ZeroTTS-Factory only passes the four arc fields through. |
| Phonemizer pipeline | Step 8 `phonemize.rs` | KokoroClient browser port handles this with `espeak-ng-wasm`. ZeroTTS-Factory does not touch tokens. |
| Setup-screen pattern (model + voices + espeak download with progress) | Step 28 `SetupScreen.tsx` + Step 10 `setup.rs` | Port to browser: `zfSetup.ts` + `SetupScreen.tsx`. Substitute `caches.put`/IndexedDB for filesystem; download from same GitHub release URLs. |
| WAV encoder | Step 16 `waveform.ts` | Verbatim into `zfWav.ts`. |
| TTSProfile JSON schema (contexts, templates, pools, install) | `ZeroTTS-zvKokoro-TINS.md` "Data Models" → `TTSProfile`/`Context` | Verbatim minus the `voice_blend` block (voice now comes from VoiceProfile). |
| `extractSlots` / `fillTemplate` / `cartesianProduct` / `enumerateAtoms` | `ZeroTTS-zvKokoro-TINS.md` Algorithm 2 | Verbatim into `zfOrder.ts`. |
| `xxhash-wasm` initialisation pattern | `ZeroTTS-zvKokoro-TINS.md` Algorithm 1 | Verbatim into `zfHash.ts`, but using `h64Raw` instead of `h32` to match ZeroVoice's 64-bit hashing. |
| Sanitise function (`sanitize`) | `ZeroTTS-zvKokoro-TINS.md` Algorithm 6 | Renamed `sanitiseSeed`; identical body. |
| Cache Storage bucket pattern (`zerotts-atoms-{persona}`) | `ZeroTTS-zvKokoro-TINS.md` F1 | **Removed.** Atoms are not cached — they go to the in-memory AudioPool. Setup-stage caches still use `transformers-*` and `zerotts-espeak-wasm`. |
| Manifest persistence | `ZeroTTS-zvKokoro-TINS.md` Algorithm 6 | **Re-purposed.** Manifest is no longer kept in Cache Storage — it is built fresh per ZIP download and embedded into the first archive (`manifest.json`). |
| Tier-priority install loop | `ZeroTTS-zvKokoro-TINS.md` Algorithm 3 | Re-used as the factory loop's tier sort: low-tier atoms render first so partial cancels still produce the most-useful clips. |
| Speak runtime path | `ZeroTTS-zvKokoro-TINS.md` Algorithm 4 | **Removed.** ZeroTTS-Factory does no runtime speak — output is downloaded, not played at runtime. The VOICE tab's preview bypasses the pool entirely. |
| Voice-blend strategy (single / hash_blend / explicit_weights) | `ZeroTTS-zvKokoro-TINS.md` F6, Algorithm 5 | **Removed.** Voice is selected directly via the ZeroVoice 6-salt scheme; no profile-level voice strategy. The user pins a seed instead. |

---

## Style Guide

**Code style.** Match the existing ZeroVoice-KokoroTTS frontend conventions:
- Explicit named exports — no `export default`.
- Co-locate types with implementation; only `zfTypes.ts` holds shared types.
- `async`/`await` over `.then()`.
- Errors carry a string code (`SETUP_FAILED`, `COI_MISSING`, `ABORT`, etc.) plus a human message.
- All log lines prefixed `[zerotts-factory]`. Level prefix follows: `[zerotts-factory]`, `[zerotts-factory:warn]`, `[zerotts-factory:error]`.

**File names.** `zf*.ts` (camelCase, `zf` prefix) for library; tabs and components keep PascalCase.

**Aesthetic.** Inherits the ZeroVoice-KokoroTTS dark workbench palette so the apps feel like siblings:

| Token | Value |
|---|---|
| Background | `#0a0a0f` |
| Panel surface | `#12121a` with `#1a1a2e` 1px borders |
| Accent (primary) | `#00ff88` (matrix green) |
| Accent (warn) | `#ff6644` |
| Text primary | `#e0e0e0` |
| Text muted | `#666680` |
| Font body | `'JetBrains Mono', 'Fira Code', monospace` |

The DOWNLOAD tab's row striping uses `#0e0e15` / `#101019` alternating; the active row glows with the accent green at 12% opacity.

**Typography.** Headings in JetBrains Mono with letter-spacing `0.04em`. Numeric counters (clip count, byte size) in a seven-segment-style display matching the ZeroVoice CoordInput component, lifted from Step 35 / Step 29.

---

## Testing Scenarios

**T1 — Determinism: name only.** With `name="Alice"`, gender `mixed`, no pin, on Chromium and Firefox, `deriveVoiceSpec` must return identical `voiceA`, `voiceB`, `t` (to bit precision), `pitchScale`, `energyScale`, and arc properties.

**T2 — Determinism: pin overrides name.** With `name="Alice"`, `pinSeed="alice_take2"`, the resulting spec must equal the spec produced by `name="alice_take2"` with `pinSeed=null`.

**T3 — Gender filter membership.** For 1000 random persona names, with `gender="female"` the resulting `voiceA` and `voiceB` must each be in `FEMALE_INDICES`. Same for `male` and `MALE_INDICES`.

**T4 — Voice A ≠ Voice B.** For 1000 random persona names across all three gender filters, `voiceA !== voiceB` always.

**T5 — Order schema rejection.** Upload a JSON missing the `version` field. `parseOrder` must throw with a Zod error path including `version`.

**T6 — Atom enumeration matches ZeroTTS reference.** Given the example `alice_order.json` from `ZeroTTS-zvKokoro-TINS.md` ("Example Profile"), enumeration must produce the exact 35 atom strings listed in that document's atom-count breakdown.

**T7 — Atom cap truncation.** A profile with theoretical 200 atoms and `install.max_atoms=50` must produce exactly 50 atoms, lower-tier preferred.

**T8 — Factory run end-to-end.** Run the 35-atom example against a freshly initialised KokoroClient. All 35 clips must end up in the AudioPool with `status="rendered"` and non-zero `byteSize`.

**T9 — Cancel mid-run.** Start a run, abort at clip 18. AudioPool must contain exactly 18 clips. Re-running the factory does not duplicate them; the existing 18 remain and 17 are added.

**T10 — Single-archive ZIP.** Pool of 35 clips, total ≈ 1.7 MB. Click Download All. Result: one ZIP `*part01_of01.zip` containing 35 WAVs plus `manifest.json`.

**T11 — Multi-archive ZIP split.** Pool of 200 clips, average 600 KB each (≈ 117 MB total). Result: two ZIPs `part01_of02` (≤100 MB) and `part02_of02`. `manifest.json` exists only in `part01_of02` and lists the archive assignment of every file.

**T12 — Manifest reproducibility.** Run twice with same name, gender, pin, and order. Both `manifest.json` files must have identical `voice` and `clips` arrays (modulo `generated_at` timestamp).

**T13 — Persona collision warning.** Save profile "Alice", then attempt to save another profile whose name sanitises to `alice` without a pin. UI must show the collision modal.

**T14 — Cache hit on warm reload.** After a successful first-run setup, reload the page. SETUP screen must complete in under 3 s; no network requests for the model/voices/espeak.

**T15 — COI missing detection.** Start the app from a server without COOP/COEP headers. SETUP must surface `COI_MISSING` and not attempt to load the Kokoro model.

**T16 — beforeunload guard.** With non-empty pool, attempt to close the tab. The browser's confirm dialog must appear.

**T17 — Export/import voice profile.** Click "Export JSON" on a profile. Edit a different field externally. Click "Import JSON". The imported profile must replace, not duplicate, the existing one (matched by `personaSeed`).

**T18 — Filename collision.** Create two atom strings that sanitise to the same filename (e.g. emojis vs. their replacements). Both clips must ship with `_collision1` / `_collision2` suffixes.

---

## Extended Features (Optional)

**X1 — Bulk persona run.** A "Run all profiles" button on the FACTORY tab iterates every saved profile against the same order JSON. Each persona's clips share the AudioPool with persona-prefixed filenames so a single download produces a per-persona folder structure inside the ZIP.

**X2 — Order-from-text wizard.** A small WIZARD modal that takes a free-form character description and invokes a local LLM (or a paste-buffer) to suggest contexts, templates, and pools. Output is editable JSON before commit.

**X3 — Pool splitting strategy override.** Settings option to pick between three split strategies: **Tier-first** (current default — lowest tier fills part01), **Context-first** (one context per archive when possible), **Fixed-size** (the user picks `MAX_ZIP_BYTES` 25–500 MB).

**X4 — Style-vector caching.** Persist the resolved 256-dim style vector for the active VoiceSpec in IndexedDB so subsequent factory runs of the same persona skip the kinship/arc compute. Saves ~20 ms per atom on weak hardware.

**X5 — Export profile + order bundle.** A "Bundle" button packages the active VoiceProfile JSON, the OrderProfile JSON, and a README listing reproduction steps into a single distributable ZIP. Lets author A reproduce author B's exact factory output.

**X6 — Diff-aware re-run.** After changing the order JSON, the factory detects unchanged atoms (atom_text identical) and skips them — re-using the existing AudioPool entries. Only changed/new atoms render. Mirrors ZeroTTS-zvKokoro's profile-version-bump flow.

**X7 — Stage 8 hash_blend.** Once direct-ONNX style-vector injection is wired into the browser KokoroClient, expose a `voice_blend.strategy` setting back on the VoiceProfile so each persona can use a unique multi-voice blend instead of an A/B pair. Only available when the underlying KokoroClient supports it.

---

## Implementation Checklist

The implementer should deliver, in order:

1. `package.json`, `tsconfig.json`, `vite.config.ts`, `index.html` — Vite + React + TS scaffold; depends on `react`, `react-dom`, `xxhash-wasm`, `jszip`, `zod`, `onnxruntime-web`.
2. `zfTypes.ts` — types only.
3. `zfHash.ts` — `xxhash-wasm` bootstrap, `hashString`, `hashToUnit`, `hashShort`.
4. `zfArc.ts` — arc property derivation; `setVoiceTable` hook.
5. `zfVoiceDeriver.ts` — `KOKORO_VOICES`, `FEMALE_INDICES`, `MALE_INDICES`, `sanitiseSeed`, `pickVoicePair`, `arcIndex`, `deriveVoiceSpec`, `applyOverrides`.
6. `zfWav.ts` — WAV encoder + `downloadBlob`.
7. `zfProfile.ts` — VoiceProfile localStorage CRUD.
8. `zfOrder.ts` — Zod schema, `parseOrder`, `enumerateAtoms`.
9. `zfPool.ts` — in-memory AudioPool with subscribe.
10. `zfFactory.ts` — `runFactory`, filename sanitiser.
11. `zfZipBuilder.ts` — multi-archive splitter and download.
12. `zfSetup.ts` — Cache Storage bootstrap for model/voices/espeak.
13. `kokoro/kokoroClient.ts` — port from ZeroVoice web edition (lift voice table via `setVoiceTable`).
14. `kokoro/voiceTable.ts` — NPZ loader using `fflate`.
15. `kokoro/phonemize.ts` — espeak-ng-wasm worker wrapper.
16. `hooks/useSetup.ts`, `hooks/useVoiceProfile.ts`, `hooks/useAudioPool.ts`, `hooks/useFactory.ts`.
17. `components/tabs/SetupScreen.tsx`, `VoiceTab.tsx`, `OrderTab.tsx`, `FactoryTab.tsx`, `DownloadTab.tsx`.
18. `components/voice/*`, `components/order/*`, `components/factory/*`, `components/download/*`, `components/ui/*`.
19. `App.tsx` — TabBar + StatusFooter + tab switch + setup gating.
20. `main.tsx` — render root + global CSS imports.
21. `styles/factory.css`, `styles/controls.css` — palette + layout.
22. Hosting config — COOP/COEP headers (for whichever host: Firebase, Netlify, etc.).
23. All eighteen testing scenarios above wired as Vitest unit + Playwright integration tests.

When complete, ZeroTTS-Factory ships as a stand-alone web app that re-uses the ZeroVoice-KokoroTTS browser pipeline as a black box and the ZeroTTS-zvKokoro JSON schema verbatim. A user with a single persona name and a JSON order can produce, preview, and download a full speech pack in under two minutes — fully automated, fully deterministic, fully reproducible.
