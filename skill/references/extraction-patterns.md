# Extraction Patterns by Input Archetype

This reference complements `SKILL.md`. It describes how to mine each input type for the raw material the order JSON needs: persona register, contexts, slots, and pool entries. Use the patterns below to drive systematic extraction rather than relying on intuition alone.

---

## Archetype A — Codebase

**Goal.** Translate the codebase's *actions* into the persona's *speech surface*. The agent in the codebase invokes tools, hits APIs, and emits errors; the persona narrates those actions.

### What to look for

| Source artifact | Maps to | Extraction technique |
|---|---|---|
| Tool registry / function-calling list | `tool_invocation` context, `{tool}` pool | Grep for `registerTool`, `tools=[`, `function:`, `@tool`, `@app.route`, FastAPI / Express route declarations |
| Error classes / error codes | `error_recovery` context, optional `{error_kind}` pool | Grep for `class .*Error`, `throw new`, `raise.*Error`, error enums |
| Status / progress hooks | `progress` context | Look for `setStatus`, `progress`, `console.log("[…]"`, telemetry emit calls |
| API endpoint names | `{tool}` pool entries | Map snake_case / camelCase identifiers to user-friendly phrases (`web_search` → "web search") |
| README persona description | persona register, name | The README often introduces the agent's voice |
| Commit messages / changelog | recent persona evolution | Optional — useful for tone calibration |

### Tool-name humanisation

Tool registries usually contain machine-readable identifiers. Convert them:

- `web_search` → "web search"
- `getCalendarEvents` → "calendar"
- `db.users.find` → "user database"
- `fs_read_file` → "file reader"

Aim for the shortest natural-language phrase a user would say aloud. Keep the pool entries phonetically distinctive — voices struggle with similar-sounding adjacent items.

### Recommended context set for codebase inputs

- **acknowledgement** (tier 1) — almost always
- **tool_invocation** (tier 1) — the core use case
- **completion** (tier 2) — when the tool returns
- **error_recovery** (tier 2) — when it doesn't
- **progress** (tier 2) — for long-running tools
- Skip **greeting** / **farewell** / **clarification** unless the agent has a chat surface beyond tool calls

### Pitfalls

- **Don't enumerate every API endpoint.** A REST API with 60 routes does not need 60 atoms. Cluster routes by user-facing concept ("search", "lookup", "save", "delete") and use the cluster names.
- **Don't include internal-only tools.** If the agent never narrates a call, the user never hears it. Ship only externally-narrated tool names.
- **Don't reuse exception class names verbatim.** `AuthenticationFailedException` is not what a persona says aloud. Translate to "couldn't sign in", "permission issue", etc.

---

## Archetype B — Plan

**Goal.** Translate the plan's *workflow* into the persona's *speech surface*. Plans describe stages, hand-offs, and decisions; each stage is a moment where the persona may speak.

### What to look for

| Source artifact | Maps to | Extraction technique |
|---|---|---|
| Numbered steps / phases | `progress` context (one template per phase) | Read step headers; pull verbs |
| Stakeholder hand-offs | `completion` context, `{recipient}` pool | "Hand to designer", "Send to QA" → "ready for {recipient}" |
| Decision points | `clarification` context | "If X then Y, else Z" → "Should I {alt_a} or {alt_b}?" |
| Named milestones | `greeting` (start of phase) and `farewell` (end of phase) | Look for "Sprint 1", "Phase Alpha", "Discovery" |
| Risk / failure modes | `error_recovery` context, `{retry_phrase}` pool | Look for "if this fails…" sections |
| Acceptance criteria | `completion` context, `{success_word}` pool | Pull adjectives — "validated", "approved", "shipped" |

### Persona register from a plan

Plans rarely describe persona voice directly, so register has to be inferred:

- Engineering plans → terse, technical, present-tense ("Building", "Shipping", "Cutting")
- Product / business plans → confident, slightly formal ("Confirmed", "Ready", "Approved")
- Creative / story plans → more decorative, possibly archaic ("Begun", "Wrought", "Set in motion")
- Game-development plans → mix of engineering and creative — pick whichever the doc leans toward

If the plan is silent on tone, ASK the user one question: "What register fits this persona — engineering / product / creative / game?"

### Recommended context set for plan inputs

- **acknowledgement** (tier 1) — start of action
- **progress** (tier 2) — phase ticks
- **completion** (tier 2) — milestone hits
- **error_recovery** (tier 2) — risk mitigation
- **clarification** (tier 3) — decision points
- Optional **greeting** / **farewell** if the plan has a clear start/end ceremony

### Pitfalls

- **Don't pull literal step titles into templates.** "Initiate Sprint 4" is not what the persona says — it's a process name. Translate to "We're starting the next sprint" or similar.
- **Don't over-context.** A plan with 30 steps does not become an order with 30 contexts. Cluster steps into 4–6 conceptual contexts.
- **Beware acronyms.** Phonemizer struggles with `KPI`, `SLA`, `MVP`. Either spell out or accept that the WAV will say each letter.

---

## Archetype C — Story

**Goal.** Translate the character's *voice* into the persona's *speech surface*. Story inputs are the richest source of register and pool material — characters in fiction are designed to sound distinctive.

### What to look for

| Source artifact | Maps to | Extraction technique |
|---|---|---|
| Character name(s) | `name`, `personaSeed` | Use the most-used name; nicknames go in pools if relevant |
| Sample dialogue | All template authoring | Read 5–15 lines; extract sentence shapes |
| Signature phrases / verbal tics | Pool entries | "He always says 'right then'" → "right then" goes in `{ack}` |
| Named entities (places, items) | `{tool}` analogue, custom pools | Story-specific vocabulary |
| Emotional beats | Context selection + tier | Stoic character → drop `error_recovery`'s apology templates |
| Vocabulary range | Register decision | Count syllables per dialogue word; archaic? clipped? technical? |
| Genre conventions | Context inclusion | Sentry → `challenge`; merchant → `pitch`; oracle → `prophecy` |

### Direct-quote extraction

If the source includes sample dialogue, use it as the *base* for at least one template per context. A character whose dialogue includes "Right, on it now" can have:

```json
"acknowledgement": {
  "tier": 1,
  "templates": [
    "{ack}, on it now.",
    "{ack}, give me a moment."
  ]
}
```

Then the `{ack}` pool draws from words that *would* fit the same sentence shape: `["Right", "Aye", "Sure thing"]`.

This is what makes story-derived orders sound authentic — the templates carry the character's actual rhythm, the pools rotate through period-appropriate variants.

### Period / setting calibration

If the source establishes a period or setting, the pool vocabulary must match:

- Medieval / fantasy → "Aye", "Forsooth", "By my troth"
- Cyberpunk → "Booted", "Spun up", "Online"
- Wild West → "Reckon so", "Mighty fine", "Much obliged"
- Modern corporate → "Confirmed", "Got it", "On it"
- Alien / inhuman → restrict consonant clusters, prefer simple syllables, avoid contractions

### Recommended context set for story inputs

This depends entirely on the character's role:

- **Companion / NPC ally** → greeting, acknowledgement, progress, farewell
- **Sentry / antagonist** → challenge (custom), threat (custom), dismiss (custom)
- **Merchant / quest-giver** → greeting, offer (custom), completion, farewell
- **Narrator** → scene_transition (custom), time_jump (custom), reveal (custom)
- **Generic chatty NPC** → all eight default contexts work fine

Define custom contexts liberally for story personas. The eight defaults are a *generic agent* surface, not a universal one.

### Pitfalls

- **Don't sanitise the character's voice.** If the source establishes that the character swears, says "mate" every line, or quotes Latin — keep it. Bowdlerised personas sound fake.
- **Don't pull obscure dialect verbatim** if it'll trip the phonemizer. Test with the Voice tab preview before committing.
- **Don't blur multi-character sources into one persona.** If the input describes three characters, ask the user which one to build the order for. Generate one file per persona.
- **Don't invent persona traits not in the source.** If the lorebook is silent on whether the character apologises, don't assume — drop `error_recovery` rather than fabricate apology lines.

---

## Hybrid sources

Real inputs are often mixed — a GDD describes a game's mechanics (codebase-like) AND its NPCs (story-like). Resolve the conflict this way:

1. **Identify which persona is being built.** A GDD might cover ten characters; you build one order at a time.
2. **For that persona, prioritise story material.** Voice trumps mechanics.
3. **Use mechanics material for tool-name pools only.** "Cast Fireball", "Open Inventory", "Save Game" go in `{tool}`-style pools because the persona narrates them.

If the input has zero story material and the persona is implicit (e.g. "the AI assistant in this app"), fall back to codebase patterns and let the user supply register on import.

---

## Cross-archetype principle

Across all three archetypes the same north star applies: **vocabulary is identity**. A pool of five words says more about the persona than any number of templates with bland filler. Spend more time on pool design than on template variety. The factory's deduplication and Cartesian product will multiply small, well-chosen pools into a coherent voice.
