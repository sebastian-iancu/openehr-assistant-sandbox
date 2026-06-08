# Refactor brief — openEHR Assistant reuse-survey & OET-authoring improvements

**Audience:** an AI refactor/feature agent working on `openehr-assistant-mcp` (the PHP MCP server) and `openehr-assistant-plugin` (the Claude Code plugin: commands, skills, agents).
**Source:** friction observed while modelling a Dutch **GGZ "Rapportage" (decursus)** template end-to-end on 2026-06-09 — delivered as a single hand-authored OET (`local/GGZ Rapportage.oet`). See the companion retrospective `docs/blog/2026-06-09-modelling-a-dutch-ggz-rapportage-with-openehr-assistant.md`.
**Status:** proposal for triage — confirm each item against the real repos before implementing.

> **Read first / de-duplicate against the sibling briefs.** Two earlier briefs already
> capture several gaps; this brief deliberately does **not** repeat them and instead
> cross-references and adds *new* evidence:
> - `docs/improvements/2026-06-09-openehr-assistant-toolchain-improvements.md` (6MWT) —
>   covers **P0-1** template/OPT compiler + `MD5-CAM`, **P0-2** `adl_validate`,
>   **P1-1** `clinical-modeler` MCP access, **P2-4** UID generation.
> - `docs/improvements/openehr-assistant-toolchain-refactor.md` (translation) — covers
>   the Dutch `language-standards-nl` guide and translation verification.
> The items below are **new**; the "Reinforcements" section at the end adds this
> session's evidence to existing items without restating them.

---

## Ground rules for the implementing agent

1. **Confirm the terrain first.** All file-path hints are *inferred* from the
   `openehr-assistant-dev` skill/agent metadata, not from direct repo reads. Run the
   `openehr-assistant-dev:repo-conventions-scout` agent (or read `AGENTS.md` /
   `CONTRIBUTING.md`) before editing.
2. **Use the dev skills**, not ad-hoc edits: `mcp-tool-authoring` (tools/resources +
   PHPUnit), `guide-prompt-authoring` (guides/prompts), `example-authoring`
   (`resources/examples/`), `agent-development`/command authoring for the plugin,
   `release-workflow` (SemVer + Keep-a-Changelog + **mcp↔plugin compatibility sync**).
3. **Additive & compatible.** New MCP tools are additive; keep the plugin's
   allowed-tool manifests in sync; bump compatibility on any signature change.
4. **Honesty over silence.** If a capability needs an external dependency (a parser, a
   national terminology/CKM endpoint), fail loudly with a clear message — never emit
   plausible-but-fake output (fabricated hashes, invented anchors).
5. **Test everything.** Each item lists acceptance criteria; turn them into PHPUnit
   tests for MCP changes.

---

## Priority 0 — the gap that broke a flagship workflow

### P0-A · Subagents are denied the MCP/CKM access their role depends on  *(HIGH — directly observed)*
- **Repos:** `openehr-assistant-plugin` (agent definitions) **and** the host
  permission/config that governs tool availability inside Task subagents.
- **Problem (observed):** The `openehr-assistant:ckm-scout` agent — whose *entire
  purpose* is an isolated, parallel CKM reuse survey — was dispatched and reported that
  **every** CKM path was denied to it: both MCP namespaces
  (`mcp__plugin_openehr-assistant_…__ckm_*` and `mcp__claude_ai_openehr-assistant_…__ckm_*`),
  `Bash` `curl`, and `WebFetch`. The **identical tools succeeded from the main session
  moments later.** The reuse-first survey — the single most-emphasised openEHR principle
  in this toolchain's own instructions — therefore had to be abandoned in the agent and
  re-run inline, which defeats the context-isolation the agent exists to provide and
  pollutes the main context with the raw search output.
- **Why it matters:** This is not a nice-to-have. `ckm-scout` is advertised
  (in its own description) as the reuse-first enforcement mechanism; if it cannot reach
  CKM in the environments where it is dispatched, the documented methodology is theatre.
  The same asymmetry will silently degrade every `Guide-First` / `Examples-First`
  subagent.
- **Proposed change:**
  1. Audit which MCP tools are exposed to Task subagents vs. the main loop, and ensure
     agents that declare CKM/guide/terminology needs (`ckm-scout`, `spec-researcher`)
     actually receive those tools in their granted set (plugin agent `tools:` /
     allowed-tool manifest).
  2. If the block is a host-level permission policy (subagents inheriting a stricter
     allowlist than the main session), document the requirement and provide a
     ready-to-merge permission snippet (e.g. project `.claude/settings.json`
     `permissions.allow` entries for the read-only `ckm_*`, `guide_*`,
     `terminology_resolve`, `examples_*` tools).
  3. Make the agents **fail loudly and early**: if a `ckm-scout` run cannot reach any
     CKM endpoint, it should return a single explicit `BLOCKED: no CKM access` status
     (it did do this well) **and** the plugin docs should tell the controller the
     supported fallback (run the survey inline) — ideally the agent itself should be
     configured so this never happens.
- **Acceptance criteria:** A `ckm-scout` dispatch in a default sandbox session completes
  a real `ckm_archetype_search` without permission denial; a regression check documents
  the exact tool grants required. No silent capability gap between main loop and
  subagent.
- **Effort/risk:** Low–Medium (mostly config/manifest + docs), very high leverage.

---

## Priority 1 — OET authoring is under-supported (archetypes are well-supported; templates are not)

### P1-A · MCP tool: authoritative **OET** schema validation (`oet_validate`)  *(HIGH — observed)*
- **Repo:** `openehr-assistant-mcp` (`src/Tools/`).
- **Problem (observed):** The only template-checking capability is the `/template-explain`
  skill — an LLM reading the file. There is **no machine validation of OET XML against
  the Ocean Template schema.** Concretely, I could not get an authoritative answer to
  "is `min=` legal on a `<Content>` element?" or "where exactly does a `/context/setting`
  constraint live?" I had to **fetch a real CKM OET (`ckm_template_get` format=oet) and
  pattern-match it**, and where uncertainty remained I downgraded a constraint to
  documentation (see P1-C) rather than risk an invalid file.
- **Relationship to existing P0-2 (`adl_validate`):** complementary, not duplicate —
  P0-2 parses *archetypes (ADL/AOM)*; this validates *templates (OET XML)*. Ideally one
  Archie/`adl2-tools`-backed dependency serves both: `adl_validate` for archetypes,
  `oet_validate`/`opt_validate` for templates.
- **Proposed change:** Add a tool that validates an OET (and OPT) against the openEHR
  template schema and returns located errors. If a real validator can't be bundled,
  at minimum validate against the OET XSD and check archetype-reference resolvability.
- **Acceptance criteria:** A deliberately invalid OET (bad attribute, unresolvable
  `archetype_id`, malformed `<Rule>` path) yields precise errors; a clean OET passes;
  `/template-explain` is updated to call it first and only explain files that validate.
- **Effort/risk:** High (external dependency) — but it removes the guesswork that this
  session ran into repeatedly.

### P1-B · MCP guide: an OET authoring/idioms reference with the **exact attribute schema**  *(HIGH — observed)*
- **Repo:** `openehr-assistant-mcp` (`resources/guides/templates/`).
- **Problem (observed):** `guide_get(templates/oet-syntax)` gives a good *overview*
  (root element, `Content`/`Items`/`Rule`, constraint types) but **not the precise,
  copy-safe attribute schema** an author needs: which attributes are valid on
  `<Content>` vs `<Item>` vs `<Rule>` (e.g. `min`/`max`/`name`/`path`/`hide_on_form`),
  how to express **occurrences on a slotted entry**, how to **fix a context value**
  (e.g. `/context/setting` to a single openEHR code), and the role of the trailing
  `<Context/>`. I reverse-engineered these from a fetched CKM OET.
- **Proposed change:** Extend `oet-syntax` (or add `templates/oet-attribute-reference`)
  with an attribute-by-element table and worked snippets for: required entry
  (`min="1" max="1"`), excluded node (`max="0"`), node rename (`name=`), inline
  `textConstraint` with `limitToList`, and **a context/setting fix-to-code idiom**.
  Cross-link from `oet-idioms-cheatsheet`.
- **Acceptance criteria:** `guide_get(templates/oet-syntax)` (or the new guide) contains
  the per-element attribute table and a verbatim "constrain `/context/setting` to a
  single openEHR `setting` code" example.
- **Effort/risk:** Low — pure content; high knowledge value. (Pairs naturally with the
  examples item P2-A.)

### P1-C · Plugin/skill: `/template-from-form` (and template-authoring) should emit a **valid OET**, not only a sketch  *(MED — from skill text + observed)*
- **Repo:** `openehr-assistant-plugin` (template-authoring skill / `/template-from-form`
  command).
- **Problem (observed):** `/template-from-form` is explicitly documented as producing a
  *design sketch, not valid OET XML*. There is no skill that takes a confirmed design to
  a **valid, slot-correct OET file**. The whole OET in this session was hand-authored
  from a guide + a fetched exemplar. Combined with P1-A/P1-B, an author has no
  "generate then validate" loop for templates the way they do for archetypes
  (`authoring` → `/archetype-lint`).
- **Proposed change:** Add a template-authoring path that emits a real OET (root
  COMPOSITION + `Content` slots + `Rule` constraints) and then calls `oet_validate`
  (P1-A) / `/template-explain`. Keep `/template-from-form` as the sketch stage feeding
  it.
- **Acceptance criteria:** From a confirmed sketch (root archetype + entry list +
  occurrences), the skill produces an OET that passes `oet_validate` and that
  `/template-explain` describes correctly.
- **Effort/risk:** Medium.

---

## Priority 2 — reuse reach & search quality (a Dutch task exposed both)

### P2-A · National / Dutch CKM & zib reuse search  *(MED — inferred; recurs across Dutch tasks)*
- **Repo:** `openehr-assistant-mcp` (`src/Tools/`), possibly config for additional CKM
  instances.
- **Problem (observed):** This was a **Dutch GGZ** task, yet the genuinely relevant
  artefacts — SOEP structures, *behandeldoel* value sets, contact-type / ZPM
  registration codes, Nictiz **zib** mappings — live in the **Dutch national CKM / zib**
  space, which `ckm_archetype_search` / `ckm_template_search` (international CKM) cannot
  reach. The assistant could only conclude "the right answer is somewhere you can't
  look." The translation brief independently hit the same national-source gap, so this
  recurs.
- **Proposed change:** Allow `ckm_*_search`/`_get` to target additional CKM instances
  (e.g. a `repository`/`instance` parameter for the openEHR NL national CKM), and/or add
  a `zib_search` tool over the Nictiz zib registry. Mark clearly when results are
  national vs. international.
- **Acceptance criteria:** A search for a Dutch-specific concept (e.g. a GGZ
  contact-type or *zib* "Alert") returns national hits, or returns a clear
  "national repository not configured" message — never a misleading empty international
  result implying the concept doesn't exist.
- **Effort/risk:** Medium–High (depends on national-CKM API access).

### P2-B · Archetype-search **recall** is lossy for exact, published concepts  *(HIGH — observed)*
- **Repo:** `openehr-assistant-mcp` (`ckm_archetype_search` query handling).
- **Problem (observed):** A search for SOAP/SOEP archetypes
  (`"SOAP subjective objective assessment plan"`, any-word) returned **no** SOAP
  archetype — yet `openEHR-EHR-SECTION.soap.v1` plainly exists and surfaced only by
  accident inside an unrelated demo OET I fetched. For a reuse-first methodology, missing
  a published, exactly-on-topic archetype is a real recall failure that pushes the model
  toward "author new" when a reuse existed.
- **Proposed change:** Investigate the search backend's recall: ensure RM-class names
  (`SECTION`, `CLUSTER`, …) and concept synonyms are indexed; consider exposing the
  `requireAllSearchWords=false` behaviour more prominently, and/or add a synonym/acronym
  map (SOAP↔SOEP). At minimum, document the recall limitation in the
  `archetype-search`/`ckm-scout` guidance so the survey widens phrasings.
- **Acceptance criteria:** `ckm_archetype_search("SOAP")` (or "soap note") returns
  `openEHR-EHR-SECTION.soap.v1` in the top results; a regression test asserts it.
- **Effort/risk:** Low–Medium (search tuning).

### P2-C · Document **this repo's** template serialization conventions  *(HIGH — observed)*
- **Repo:** `openehr-assistant-mcp` (`resources/guides/templates/`) and/or the sandbox
  repo's `CLAUDE.md`.
- **Problem (observed):** The sandbox's `*.t.json` files are **ADL Designer
  differential-template JSON** (`@type:TEMPLATE`, `parentArchetypeId`, a `t_<concept>`
  archetype id, `C_COMPLEX_OBJECT` definition, `templateOverlays`, tool-generated
  `MD5-CAM`/`buildUid`). The project's own `CLAUDE.md` calls them "openEHR Web Template
  (simplified) JSON" — a **different format**. This mislabel cost real time
  (I assumed a Better/EHRbase web template) and directly contributed to dropping the
  `.t.json` deliverable. The Vital signs `.t.json` is also effectively a stub (empty
  `content`), which is misleading as an exemplar.
- **Proposed change:** Add a short guide (`templates/serialization-formats`) that
  distinguishes **OET** (design-time XML), **OPT** (operational XML), **ADL Designer
  differential JSON** (`.t.json` here), and **Better web template JSON** — when each is
  hand-authorable vs tool-generated, and which checksums each carries. Correct the
  sandbox `CLAUDE.md` description.
- **Acceptance criteria:** A new guide enumerates the four formats with a
  "hand-authorable? / carries tool hashes?" matrix; the sandbox `CLAUDE.md` template
  description is accurate.
- **Effort/risk:** Low.

---

## Priority 3 — observations (host/config, likely out of repo scope)

### P3-A · Dual MCP namespace is confusing and one half dropped mid-session  *(HIGH — observed; flag only)*
- The same tools were exposed under both `mcp__plugin_openehr-assistant_openehr-assistant__*`
  and `mcp__claude_ai_openehr-assistant__*`. The `claude_ai` server **disconnected
  partway through** (its tools became unavailable); everything continued via the
  `plugin` namespace. The redundancy is a foot-gun: it's non-obvious which to call, and a
  blocked/disconnected namespace reads as a capability gap rather than a routing issue.
  This is primarily a **host/MCP-config** concern, but worth a note in the plugin README
  ("prefer the `plugin` namespace; the `claude_ai` connector is optional/duplicative").
- **Action:** documentation + guidance; no code change in the two repos unless the
  plugin itself registers both.

---

## Reinforcements (this session adds evidence — see sibling briefs, do not duplicate)

- **6MWT P0-1 (template/OPT compiler + `MD5-CAM`/`buildUid` generation).** Reinforced and
  *causal* here: the absence of any hash generator is exactly why the user dropped the
  `.t.json` deliverable and the session shipped OET-only. Second independent occurrence →
  raise priority.
- **6MWT P1-1 (`clinical-modeler` lacks MCP access).** Reinforced: the implementer for
  this task had to be a `general-purpose` agent specifically so it could run the
  `/template-explain` skill — `clinical-modeler` could not. Combine with **P0-A** above:
  the fix is the same family (right tools to the right agents).
- **6MWT P2-4 (portable UID generation).** Reinforced: `uuidgen` was again absent;
  worked around via `python3 -c 'import uuid'`.

---

## Suggested sequencing

1. **P0-A** first — cheap config/manifest + docs, unblocks the reuse-first methodology.
2. **P1-B** and **P2-C** — pure content, high leverage, no dependencies.
3. **P2-B** — search recall tuning + regression test.
4. **P1-C** — template-authoring → valid-OET path.
5. **P1-A** / **P2-A** — once an external dependency strategy (Archie validator;
   national-CKM endpoint) is agreed; coordinate with 6MWT **P0-2**.
6. Bundle the **6MWT P0-1** hash/OPT work given its second occurrence.

Each shipped item: bump versions and reconcile mcp↔plugin compatibility via
`openehr-assistant-dev:release-workflow`, and update both changelogs.

## Open questions to resolve before coding

- Is P0-A a plugin agent `tools:` grant issue, a host permission policy, or both?
  (Reproduce a `ckm-scout` dispatch and inspect the denial source.)
- Can the MCP target the Dutch national CKM / Nictiz zib registry at all (API/auth)?
  (gates P2-A)
- Is there a bundlable openEHR validator (Archie / adl2-tools) for the PHP server, or
  must validation shell out? (gates P1-A and 6MWT P0-2)
- Confirm real paths for the plugin command/agent files and the MCP `src/Tools/` +
  `resources/guides/templates/` dirs.
