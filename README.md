# ZeroTTS-Factory

**Browser-based deterministic speech factory.** Drop in a JSON order file, pick a persona name + gender + seed, click run — get back a ZIP of WAV files for every line your character can say. All inference runs locally in the browser via Kokoro WASM. No server. No API keys. Reproducible across machines.

```
   persona name   ─┐
   gender filter   ├──xxh64──►  VoiceSpec (voice A, voice B, blend t,
   pin seed (42)  ─┘            pitch, energy, arc properties)
                                       │
   JSON order ──► atom enumeration ──► Kokoro WASM ──► WAV pool
                  (templates × pools)                   (download as ZIP)
```

Live build: <https://www.scuffedepoch.com/zerotts-factory/>

---

## What this is good for

- **Voice packs for game NPCs** — dial in 30–80 stock lines per character, ship the WAVs.
- **Pre-rendered chat-agent fillers** — acknowledgements, tool-invocation narration, error recovery, progress checks. Cached audio means zero perceived latency at runtime.
- **Reproducible character voices** — the (name, gender, seed) tuple uniquely identifies a voice. Two authors with the same triple get byte-comparable audio without sharing files.
- **Offline-first speech generation** — model and voices cache to the browser's persistent storage. After first setup, everything runs without network.

---

## Key features

- **Persona-name as voice seed** (string-as-seed, ported from ZeroVoice-KokoroTTS' position-as-seed scheme). Six independent salts generate voice A/B indices, blend t, pitch, energy, plus four Zero-Quadratic arc properties.
- **Three-axis reproduction tuple** — name + gender + pin seed. All three are visible in the UI; change any one to dial a different voice.
- **JSON order schema** — contexts, templates, slot tokens, shared pools, tier priorities, atom-count cap. Matches the upstream ZeroTTS-zvKokoro schema (minus the voice-blend block).
- **Cartesian-product atom enumeration** with deduplication and tier-priority truncation under the cap.
- **Multi-archive ZIP download** — caps each archive at 100 MB and writes a JSON manifest into part 01 listing every clip's archive assignment, voice spec, and order version.
- **Hugging Face mirror integration** — Kokoro model and 26 voice files pulled from `onnx-community/Kokoro-82M-v1.0-ONNX` (CORS-friendly, no proxy required).
- **Cross-origin-isolated WASM** — full Kokoro WASM-SIMD inference with `crossOriginIsolated: true` headers.
- **Apache `.htaccess` config** included for one-shot deploy.
- **Authoring skill** for Claude Code — drop a codebase, plan, or story and get a complete order.json back.

---

## Quick start (development)

```bash
git clone https://github.com/MushroomFleet/zerotts-factory-dev.git
cd zerotts-factory-dev
npm install
npm run dev
```

Open <http://localhost:5173/zerotts-factory/>. The first load runs through SETUP — model + 26 voice files (~99 MB total) cache to your browser. Subsequent loads complete in under 3 seconds.

### Requirements

- Node 18+ (for Vite 5).
- Chrome / Edge / Firefox (Safari is best-effort — `navigator.storage.persist()` prompts user).
- The dev server sets `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: credentialless` automatically.

---

## How it works

### Voice derivation

```
seed   = `${personaSeed}|${pinSeed}`        // e.g. "alice|42"
hashes = xxh64(seed, salt)  for salt in {VOICE_A, VOICE_B, T, PITCH, ENERGY}
indices apply gender filter (mixed=26 / female=14 / male=12)
arc properties = pair-symmetric xxh64 over (voiceA, voiceB) — Zero-Quadratic
```

Same triple anywhere in the world gives the same VoiceSpec. The 64-bit hash space and 26-voice base × float32 blend resolution gives ~10^18 effective unique voices.

### Order pipeline

```
Order JSON   ─►  Zod schema validation
             ─►  Cartesian product per template (slot fills × pool entries)
             ─►  Dedupe by atom text
             ─►  Tier-sorted enumeration (1 → 2 → 3)
             ─►  Per-atom Kokoro inference
             ─►  Float32 PCM → WAV blob
             ─►  In-memory AudioPool
             ─►  Multi-archive ZIP build (≤100 MB each, manifest in part 01)
```

Defaults to a 100-atom cap (~50 s render time, ~5 MB pool size). Cap is raisable in the order JSON.

---

## Building and deploying

```bash
npm run build
```

Output is in `dist/`. Includes a production-ready `.htaccess` for Apache hosts.

### Apache (recommended)

1. Upload `dist/*` (including `.htaccess`) to your `<base>/zerotts-factory/` path.
2. Confirm `mod_headers` and `mod_mime` are enabled on the host.
3. The `.htaccess` sets:
   - `Cross-Origin-Opener-Policy: same-origin`
   - `Cross-Origin-Embedder-Policy: credentialless`
   - `Cross-Origin-Resource-Policy: same-origin`
   - `application/wasm` MIME for `.wasm` files
   - Long-cache for hashed assets, no-cache for `.html`
   - SPA fallback that **does not** rewrite asset paths (so missing assets return real 404s instead of silent index.html corruption)

### Other hosts (Netlify, Cloudflare Pages, Firebase Hosting, etc.)

Configure the equivalent COOP/COEP/CORP headers and the `application/wasm` MIME type. Deploy `dist/` at a `/zerotts-factory/` base path (or change `base` in `vite.config.ts`).

### Self-hosting the model + voices

By default, model and voices are pulled from Hugging Face's `onnx-community/Kokoro-82M-v1.0-ONNX`. To self-host:

1. Mirror `onnx/model_q8f16.onnx` and the contents of `voices/` to your CDN.
2. Set `VITE_ASSET_BASE_URL` in `.env.production` to your CDN base (no trailing slash).
3. The deployer is responsible for serving with `Access-Control-Allow-Origin: *` (or same-origin to the app).

---

## Project structure

```
.
├── src/
│   ├── lib/                          # core (no DOM):
│   │   ├── zfTypes.ts                #   shared types
│   │   ├── zfHash.ts                 #   xxhash-wasm wrapper
│   │   ├── zfArc.ts                  #   Zero-Quadratic arc derivation
│   │   ├── zfVoiceDeriver.ts         #   name+seed -> VoiceSpec
│   │   ├── zfProfile.ts              #   localStorage CRUD + migration
│   │   ├── zfOrder.ts                #   Zod schema + atom enumeration
│   │   ├── zfPool.ts                 #   in-memory AudioPool
│   │   ├── zfFactory.ts              #   batch render loop
│   │   ├── zfZipBuilder.ts           #   multi-archive splitter
│   │   ├── zfWav.ts                  #   PCM -> WAV encoder
│   │   └── zfSetup.ts                #   asset bootstrap (HF mirror)
│   ├── kokoro/
│   │   ├── kokoroClient.ts           #   onnxruntime-web wrapper
│   │   ├── voiceTable.ts             #   per-voice byte assembler
│   │   └── phonemize.ts              #   espeak-ng-wasm wrapper
│   ├── hooks/                        # useSetup / useFactory / etc.
│   ├── components/                   # tabs + sub-components
│   └── styles/                       # factory.css + controls.css
├── public/
│   └── .htaccess                     # production Apache config
├── examples/
│   └── example-character-order.json  # reference order
├── docs/
│   ├── order-authoring-guide.md      # how to write an order JSON
│   └── stage1-qa-fixes.md            # changelog / design notes
├── skill/
│   └── Zerotts-factory-builder.skill/ # Claude Code skill for order generation
└── ZeroTTS-Factory-TINS.md           # full TINS specification
```

---

## Documentation

- **[`ZeroTTS-Factory-TINS.md`](./ZeroTTS-Factory-TINS.md)** — full system specification (TINS = "There Is No Source"; the README is so detailed any capable LLM can regenerate the implementation).
- **[`docs/order-authoring-guide.md`](./docs/order-authoring-guide.md)** — how to write an order JSON: schema, slot syntax, tier strategy, pool design, atom-count math.
- **[`examples/example-character-order.json`](./examples/example-character-order.json)** — reference order with 58 atoms across 8 contexts.
- **[`docs/stage1-qa-fixes.md`](./docs/stage1-qa-fixes.md)** — design notes on the v0.2.0 QA-driven UX changes.
- **[`skill/Zerotts-factory-builder.skill/`](./skill/Zerotts-factory-builder.skill/)** — Claude Code skill that generates order JSONs from codebases / plans / stories.

### Upstream specifications

- **[`ZeroVoice-KokoroTTS-postV2-TINS.md`](./ZeroVoice-KokoroTTS-postV2-TINS.md)** — the desktop workbench this project's voice derivation is ported from.
- **[`ZeroTTS-zvKokoro-TINS.md`](./ZeroTTS-zvKokoro-TINS.md)** — the cached-speech runtime layer this project's order schema is adapted from.

---

## Acknowledgements

- **Kokoro-82M** — the underlying TTS model (hexgrad / community).
- **onnx-community/Kokoro-82M-v1.0-ONNX** — the CORS-friendly HF mirror this build pulls from.
- **`onnxruntime-web`** — browser-side ONNX inference.
- **`phonemizer`** (xenova) — espeak-ng-wasm wrapper for IPA generation.
- **`xxhash-wasm`** — fast, deterministic, browser-compatible hashing.
- **`jszip`** — multi-archive ZIP construction.

The position-as-seed determinism scheme and Zero-Quadratic arc properties are from the **ZeroVoice-KokoroTTS** workbench. The atom-enumeration / template-pool model is from **ZeroTTS-zvKokoro**. ZeroTTS-Factory adapts both into a batch-render pipeline.

---

## Contributing

Issues and PRs welcome. The codebase aims to:

- Stay pure-browser (no server, no build-step model conversion).
- Match upstream schemas where reasonable, diverge clearly where not.
- Surface deterministic, reproducible audio as the primary value.
- Trust the engine — UI defaults to hidden engine internals (override locks were removed in v0.2.0 per QA).

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{zerotts_factory_2025,
  title   = {ZeroTTS-Factory: Browser-based deterministic speech factory built on Kokoro WASM TTS},
  author  = {Drift Johnson},
  year    = {2025},
  url     = {https://github.com/MushroomFleet/zerotts-factory},
  version = {0.2.0}
}
```

### Donate:


[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
