---
name: zerotts-factory-builder
description: Generates complete order.json files for the ZeroTTS-Factory speech-generation
  system from a codebase, an implementation plan, or a story/character document.
  Use this skill when the user asks to "create an order JSON", "build a speech
  order", "generate factory input", "make atoms for a character", or presents
  any codebase / plan / narrative source and asks for a ZeroTTS-Factory order.
---

# Zerotts-factory-builder

You generate **order.json** files for the ZeroTTS-Factory pipeline. The user provides a source — a codebase, a written plan, or a story / character document — and you turn it into a JSON order: contexts, templates, pools, and tier assignments that map onto the speech surface implied by the source.

Two project files are authoritative. **Read them first, every time, before writing.** They are not redundant — the example is the structural prior; the guide is the schema and rationale.

- **Template (structural prior):** `references/example-character-order.json`
- **Authoring guide (schema + rationale):** `references/order-authoring-guide.md`
- **Companion reference (extraction patterns by input type):** `references/extraction-patterns.md`

## Trigger conditions

Run this skill when the user does any of the following:

- Provides a codebase / repo path and asks for a ZeroTTS / speech-factory order.
- Provides a story, character bible, lorebook, or narrative document and asks for one.
- Provides an implementation plan, GDD, or workflow doc and asks for one.
- Asks to "build", "create", "generate", or "write" an `order.json` for the factory.
- Asks to "make atoms" or "create speech atoms" for a named character.

Do **not** run when:

- The user asks about the *Voice profile* JSON (that artifact is created in the app's Voice tab — different schema, different purpose).
- The user is editing an existing order JSON manually and only needs syntax help.

## Operating procedure

Follow these steps in order. Do not skip the bootstrap.

### Step 1 — Bootstrap

Read in this exact order, then internalise:

1. `references/order-authoring-guide.md` — the schema reference, tier semantics, atom-count math, and the 100-atom cap rationale.
2. `references/example-character-order.json` — the canonical structural prior.
3. `references/extraction-patterns.md` — patterns specific to codebase / plan / story inputs.

If any of these files cannot be read, STOP and tell the user — do not proceed from training-data memory of the schema, because the schema may have changed.

### Step 2 — Classify the input

Decide which of the three input archetypes applies. The extraction patterns differ.

| Archetype | Signals | Primary extraction targets |
|---|---|---|
| **codebase** | Repo path, package.json / pyproject / Cargo.toml, source files, tool registries, error messages | Tool names, API endpoint names, error categories, status messages |
| **plan** | Markdown spec, GDD, workflow doc, implementation plan, RFC, ticket | Agent roles, lifecycle stages, hand-off points, named operations |
| **story** | Lorebook, character bible, screenplay, narrative prose, persona dossier | Character name(s), register, signature phrases, emotional beats, recurring situations |

If the input is mixed, treat the dominant signal as the archetype and note minor sources as supplementary pools.

### Step 3 — Identify the persona

The order JSON is voice-agnostic — voice is set by a separate VoiceProfile in the app. But the *vocabulary* of the order must match a persona. Determine:

1. **Persona name** — what to call the file (`<persona_seed>-order.json`).
2. **Register** — formal / casual / technical / archaic / clipped. This drives every pool fill.
3. **Domain** — what the persona talks about. Tools, locations, named entities.
4. **Emotional range** — does this persona apologise? Get excited? Stay deadpan? Drives `prosody_tag` and which contexts to include.

If two of these four are unclear from the source, ASK the user **one** focused question to resolve them. Do not ask three questions in a row — pick the most blocking gap and ask only that.

### Step 4 — Design the context set

The example covers eight standard contexts: `greeting`, `acknowledgement`, `tool_invocation`, `progress`, `completion`, `error_recovery`, `clarification`, `farewell`. These are a **default surface** — keep, drop, or add based on the persona.

| Persona type | Likely keep | Likely drop | Likely add |
|---|---|---|---|
| Customer-support bot | all eight | — | `escalation`, `policy_quote` |
| Game NPC sentry | acknowledgement, error_recovery, farewell | greeting, completion, clarification | `alert`, `challenge`, `dismiss` |
| Narrator (story) | greeting, progress, farewell | tool_invocation, error_recovery | `scene_transition`, `time_jump`, `omen` |
| Tool-heavy agent | acknowledgement, tool_invocation, completion, error_recovery | farewell, clarification | `tool_disambiguation`, `permission_prompt` |
| Ambient companion | greeting, progress, farewell | tool_invocation, error_recovery | `idle_chatter`, `affirmation`, `humming` |

Tier assignment:

- **Tier 1** — what the user hears constantly. ~30–60% of total atoms.
- **Tier 2** — common but not load-bearing.
- **Tier 3** — rare, conversational corners. Render last; first to be evicted under cap.

### Step 5 — Build pools

For each slot a context needs, build a pool. Rules:

- **3–8 entries** per pool typical. The Cartesian product is multiplicative.
- **Match the register** — every pool entry sounds like the persona.
- **Cover every external value** — if `{tool}` is in a template, every tool the agent can call must appear in the pool. Pull tool names directly from the source codebase (look for tool registries, `registerTool` calls, FastAPI route names, etc.).
- **Share pools across contexts** when vocabulary overlaps — e.g. `{ack}` used by both `acknowledgement` and `tool_invocation` in the example.
- **Avoid `{` or `}` inside pool values** — they collide with slot syntax.

### Step 6 — Compute the atom count

Before writing, count atoms exactly. Use the formula per template:

```
atoms_per_template = product of (pool_size for each slot)
                   = 1 if template has no slots
```

Sum across templates within a context, sum across contexts.

If the total exceeds **100** (default cap), do one of:

1. **Trim pools.** Most common fix — multiplicative effect.
2. **Split templates.** Replace one 3-slot template with two 2-slot templates that don't overlap.
3. **Drop a tier-3 context** entirely.
4. **Raise `install.max_atoms`** — only with explicit user approval, and only if the use case warrants it. Default is 100 for a reason: ~50 s render time at 0.5 s/atom.

Show the count breakdown in your final response so the user can sanity-check.

### Step 7 — Write the file

Output a single JSON file. Conventions:

- Filename: `<persona_seed>-order.json` where `persona_seed` is the lowercased, sanitised persona name (`[^a-z0-9_-]` → `_`). E.g. "Captain Pike" → `captain_pike-order.json`.
- Top-level fields in this order: `name`, `description`, `version`, `contexts`, `pools`, `install`.
- `version`: start at `"1.0.0"`.
- `description`: one or two sentences naming the persona, the source archetype, and the atom count.
- Pretty-print (2-space indent).
- No trailing comments — JSON does not support them. The schema rejects unknown top-level fields.
- Keep slot names `[A-Za-z0-9_]` only.

Default location: `examples/<persona_seed>-order.json` unless the user specifies a different path.

### Step 8 — Validate before claiming done

Mentally walk through the Zod schema in `src/lib/zfOrder.ts`:

- [ ] `name`, `version` are non-empty strings.
- [ ] `contexts` has at least one entry.
- [ ] Every `contexts[].templates` array is non-empty.
- [ ] Every `contexts[].tier` is `1`, `2`, or `3`.
- [ ] Every `pools[]` array is non-empty.
- [ ] Every slot `{name}` referenced by a template has a corresponding `pools` entry.
- [ ] `install.max_atoms` (if set) is a positive integer ≤ 10000.
- [ ] No template renders to an empty string.
- [ ] No `{` or `}` characters inside pool values.

### Step 9 — Report

End with a short summary that lets the user decide whether to ship the file:

```
Generated <filename>
- Persona: <name> (<register>, <domain>)
- Contexts: <N> (T1: <a>, T2: <b>, T3: <c>)
- Pools: <M>
- Atoms: <total> / <cap>
  - <context>: <count>
  - ...
- Source: <archetype>
- Suggested next step: drop into Order tab, set up matching VoiceProfile (gender: <suggestion>)
```

Suggest a gender filter (`mixed` / `female` / `male`) only if the source clearly implies one. Otherwise say "gender filter: user choice".

## Anti-patterns

- **Don't invent tool names** for a codebase input. Read the source. If you can't find them, ask.
- **Don't pad pools** to feel comprehensive. A 4-entry pool that all sound like the persona beats a 12-entry pool with three-quarter filler.
- **Don't ship over the cap silently.** If you have to truncate, surface it in the report.
- **Don't write multiple files in one invocation** unless the user explicitly asks for multiple personas. One persona, one file.
- **Don't reuse the example's pools verbatim** unless the persona genuinely is a "generic agent". The example is a *shape*, not a vocabulary source.
- **Don't add `voice_blend`, `runtime_requirements`, or other fields** from the upstream ZeroTTS-zvKokoro schema. Those are not part of the ZeroTTS-Factory order schema. Reading the guide will make this clear.
