# ZeroTTS-Factory — Order Authoring Guide

This guide explains how to write the JSON file that drives the speech factory. The companion file [`examples/example-character-order.json`](../examples/example-character-order.json) is a working starting point — copy it, rename it, and edit.

---

## 1. What an order JSON is

A **voice profile** is created in the app (Voice tab). It carries the persona name, gender filter, optional pin seed, and any locked overrides. It decides *how the voice sounds*.

An **order JSON** is the file you upload. It carries the speech surface — the contexts, sentence templates, and word pools — that the factory expands into individual atoms. It decides *what the voice says*.

Think of it as a recipe: the templates are sentence shapes with `{slot}` holes; the pools are the lists of phrases that fill those holes. The factory enumerates every legal combination and synthesises one WAV file per resulting sentence.

---

## 2. Quick start

The smallest valid order:

```json
{
  "name": "Hello world",
  "version": "1.0.0",
  "contexts": {
    "greeting": {
      "tier": 1,
      "templates": ["Hello, {name}."]
    }
  },
  "pools": {
    "name": ["world", "friend"]
  },
  "install": {}
}
```

This produces two atoms:
- `Hello, world.`
- `Hello, friend.`

Drop it on the **Order** tab, switch to **Factory**, click **Run Factory**. Two clips appear on the **Download** tab.

---

## 3. Schema reference

```jsonc
{
  "name": "string",                  // human label, shown in UI
  "description": "string",            // optional, shown under metadata
  "version": "string",                // semver, e.g. "1.0.0" — your bookkeeping
  "contexts": {                       // map of context name → context body
    "<context_name>": {
      "tier": 1 | 2 | 3,              // priority — see §5
      "prosody_tag": "string",        // optional, hint only — not used by factory
      "trigger_hints": ["string"],    // optional, notes for human authors
      "templates": ["string"]         // sentence shapes with {slot} tokens
    }
  },
  "pools": {                          // map of pool name → list of fill values
    "<pool_name>": ["string", ...]
  },
  "install": {
    "max_atoms": 100                  // optional cap, default 100
  }
}
```

### Allowed field summary

| Field | Required | Type | Notes |
|---|---|---|---|
| `name` | yes | string | Used in archive filenames after slug-sanitisation |
| `description` | no | string | UI only |
| `version` | yes | string | Authoring discipline; recorded in manifest |
| `contexts` | yes | object | At least one context required by Zod |
| `contexts[].tier` | yes | `1` \| `2` \| `3` | Lower = higher priority |
| `contexts[].templates` | yes | array of strings | At least one template |
| `contexts[].prosody_tag` | no | string | Free-text hint, e.g. "confident", "apologetic" |
| `contexts[].trigger_hints` | no | array of strings | Notes for fellow authors |
| `pools` | yes | object | Map of pool name → at least one entry |
| `install.max_atoms` | no | int | Defaults to 100, max 10000 |

The schema is enforced at upload time by Zod. A bad file gets you a precise error path so you can fix it.

---

## 4. The atom enumeration model

### Slot syntax

Inside a template, `{name}` is a **slot**. The slot name must match a key in `pools`. Slot names use `[A-Za-z0-9_]`.

```
"templates": ["I'll use the {tool} now."]
"pools":    { "tool": ["web search", "calculator"] }
```

→ enumerates:
- `I'll use the web search now.`
- `I'll use the calculator now.`

### Multiple slots — Cartesian product

If a template has more than one slot, the factory enumerates **every combination**:

```
"templates": ["{ack}, reaching for the {tool}."]
"pools":    { "ack": ["Sure", "Right"], "tool": ["search", "calculator"] }
```

→ enumerates 2 × 2 = 4 atoms:
- `Sure, reaching for the search.`
- `Sure, reaching for the calculator.`
- `Right, reaching for the search.`
- `Right, reaching for the calculator.`

This grows multiplicatively. A template with three 5-entry pools produces 125 atoms by itself. Watch the count.

### Slot-less templates

A template with no slots is its own atom:

```
"templates": ["Nearly there."]
```

→ 1 atom. Ideal for **progress** or **farewell** contexts where the line doesn't vary.

### Deduplication

If two templates resolve to the same string, the factory keeps only one. So `["Hello.", "Hello."]` produces 1 atom, not 2.

### Pools shared across contexts

You can re-use a pool from any number of contexts. The example file does this with `{ack}` (used by both `acknowledgement` and `tool_invocation`). Sharing keeps your vocabulary tight and consistent.

---

## 5. Tier strategy

Every context has a `tier` of `1`, `2`, or `3`. Tier orders the *render queue* — tier-1 atoms render first, then tier-2, then tier-3. If you cancel mid-run, you keep the most-important atoms.

| Tier | Meaning | Examples |
|---|---|---|
| **1** | Persona-defining; users hear it constantly | greetings, acknowledgements, tool invocations |
| **2** | Common but not critical | progress checks, completion, error recovery |
| **3** | Rare; conversational corners | clarification, farewells, niche emotional beats |

Tier also drives the `install.max_atoms` truncation: when the cap is hit, lower-tier atoms are evicted first because the enumeration walks contexts tier-ascending.

**Practical advice.** Keep tier-1 lean — typically 30–60% of your total atoms. Most of the listening time is here, so render quality and variation matter most.

---

## 6. Pool design tips

### Keep pools tight (3–8 entries)

The Cartesian product magnifies pool size. A 5×4 template is usually plenty of variation. If you need more, add another *template shape* rather than expanding pools.

### Match the persona's register

If the persona is formal, fill `{ack}` with `["Certainly", "Of course", "Indeed"]`. If casual, `["Yeah", "Sure", "Cool"]`. Mismatched register sounds uncanny.

### Cover every external value

If you use a `{tool}` slot, the pool must contain *every* tool name your agent can invoke. The runtime can't synthesise a value that wasn't pre-rendered. The factory warns if a template references a missing pool.

### Avoid overlapping vocabulary across pools

If `pool_a` and `pool_b` both contain "okay", their meanings get confused at authoring time. Better to merge them into a single pool when possible.

### Consider phonetic distinctiveness

The TTS engine handles common English well, but pool entries with unusual punctuation, diacritics, or technical jargon may render poorly. The factory will mark such clips with a yellow ⚠ when phonemizer characters are skipped. Test edge cases with the **Voice** tab preview before committing.

---

## 7. The 100-atom cap

`install.max_atoms` defaults to 100. You can raise it up to 10,000 in the schema, but understand the trade-off:

- Each atom costs ~0.5 seconds to synthesise on a 2020-era laptop.
- 100 atoms ≈ 50 s; 1000 atoms ≈ 8 minutes; 10,000 atoms ≈ 80 minutes.
- Each WAV averages ~50 KB. 100 atoms ≈ 5 MB; 10,000 atoms ≈ 500 MB (will split into 5 ZIP archives).

If your enumeration overflows the cap, a **truncated** banner appears on the Order tab and the factory only renders the first `max_atoms` (lowest tier first). You have three options:

1. **Trim pools** — the most common fix. Pool size has multiplicative effect.
2. **Split templates** — instead of one template with 3 slots, write two templates with 2 slots each.
3. **Raise the cap** — only if you really need that many lines. Most personas need 30–80 atoms; production ChatGPT-class personas land around 150–250.

### Sanity numbers from the example

The reference order produces **58 atoms** total:

| Context | Templates × Pool sizes | Atoms |
|---|---|---|
| greeting | 1 × 3 × 2 | 6 |
| acknowledgement | 2 × 5 | 10 |
| tool_invocation | 1×4 + 5×4 | 24 |
| progress | 4 (no slots) | 4 |
| completion | 1 × 3 × 2 | 6 |
| error_recovery | 1 × 2 | 2 |
| clarification | 1 × 3 | 3 |
| farewell | 3 (no slots) | 3 |
| **total** | | **58** |

That's a comfortable surface area for a generic agent persona. Use it as a yardstick.

---

## 8. Reproducibility

The factory is deterministic in three ways:

1. **Voice derivation** — given the same `personaSeed` (or pin), gender, and (locked) overrides, the voice spec is bit-identical across machines.
2. **Atom enumeration** — given the same order JSON, atoms are produced in the same order with the same internal IDs.
3. **WAV output** — Kokoro's WASM inference is deterministic given the same tokens and style vector.

Practical consequence: if you commit the order JSON to source control alongside the voice profile JSON, anyone can regenerate the exact pack of WAVs. Bundle both files together when you ship a character.

---

## 9. Versioning your order

Bump `version` whenever you change templates or pools. The factory does not currently use this field for diff-aware re-runs, but the manifest embedded in the downloaded ZIP records it — useful when you ship a v1.1 of a character pack and want to know what changed.

A reasonable convention:

| Change | Bump |
|---|---|
| Typo fix in a single template | patch — `1.0.0` → `1.0.1` |
| Adding a new template within an existing context | minor — `1.0.0` → `1.1.0` |
| Adding/renaming a context, pool, or slot | major — `1.0.0` → `2.0.0` |
| Removing a context | major |

---

## 10. Common pitfalls

**A template references a slot you forgot to declare.**
You'll see `[zerotts-factory] template "I love {color}" missing pool entries — skipped` in the browser console, and the template silently disappears from enumeration. Add the pool, or rename the slot.

**Empty pool array.**
Zod rejects with a path like `pools.tool` and the message `Array must contain at least 1 element(s)`. Fill it.

**Identical atom strings produce one clip, not many.**
This is by design (deduplication). If you want multiple takes of the same line, use different templates: `"Hello."` and `"Hello there."` — not two `"Hello."` entries.

**Missing tool names in `{tool}` pool.**
At runtime the agent will play silence (or fall back to live synthesis if you've wired that up). Keep the pool synchronised with the agent's tool registry.

**Pool entries with curly braces.**
`"{": "{"`-style entries break slot detection. Don't put `{` or `}` inside pool values; they collide with the slot syntax.

**File too large for `localStorage`.**
The order JSON itself is uploaded ad-hoc and not persisted, so size is not constrained by `localStorage`. But the rendered AudioPool lives in memory — 1000+ atoms can pressure browser memory. Build incrementally and download in batches.

---

## 11. The example as a launchpad

`examples/example-character-order.json` is engineered to demonstrate every feature without overwhelming. The recommended workflow:

1. **Copy** it to `my-character-order.json`.
2. **Rename** the `name` and `description` fields.
3. **Read every context** and decide which to keep, edit, or drop.
4. **Edit the pools** to match your character's voice — vocabulary is identity.
5. **Adjust tiers** if your persona's priorities differ (e.g. a customer-support bot has tier-1 acknowledgements but tier-2 farewells; a NPC sentry might invert this).
6. **Bump `version`** and start a fresh `CHANGELOG` per file.
7. **Test** with a 5-atom subset first (set `max_atoms: 5`) so you hear what the persona sounds like before committing to a full 60-atom run.
8. **Iterate** — voice, tier, and pool tuning is fast because each round is < 60 seconds.

---

## 12. Outside this file

- **Voice profile JSON** (created in the **Voice** tab) is a separate artifact. Export it to capture the persona name, gender, pin seed, and any locked overrides. Ship it alongside the order JSON.
- **The manifest** generated by **Download All (ZIP)** records every clip, archive assignment, voice spec, and order version. It is a complete reproduction record — keep it.
- **The TINS spec** ([`ZeroTTS-Factory-TINS.md`](../ZeroTTS-Factory-TINS.md)) is the authoritative reference for the system's behaviour. Read it if you want to understand *why* the schema looks the way it does.
