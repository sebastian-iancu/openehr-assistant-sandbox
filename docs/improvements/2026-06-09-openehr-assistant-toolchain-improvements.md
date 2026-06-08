# Refactor brief — openEHR Assistant toolchain improvements

**Audience:** an AI refactor/feature agent working on `openehr-assistant-mcp` (the MCP server) and `openehr-assistant-plugin` (the Claude Code plugin).
**Source:** friction observed while modelling a 6-Minute Walk Test end-to-end (archetype + template) using these tools on 2026-06-09. See the companion retrospective `docs/blog/2026-06-09-modelling-the-six-minute-walk-test-with-openehr-assistant.md`.
**Status:** proposal for triage — do not implement blindly; confirm each item against the real repos first.

---

## Ground rules for the implementing agent

1. **Confirm the terrain before touching it.** The file-path hints below are *inferred* from the `openehr-assistant-dev` plugin's skill descriptions, not from direct inspection of the repos. Before editing, run the `openehr-assistant-dev:repo-conventions-scout` agent (or read `AGENTS.md`/`CONTRIBUTING.md`) to confirm the actual layout, the MCP tool-registration pattern (`#[McpTool]` etc.), and the resource conventions.
2. **Use the dev skills.** `openehr-assistant-dev:mcp-tool-authoring` (new tools/resources + PHPUnit), `guide-prompt-authoring` (guides/prompts), `example-authoring` (the `resources/examples/` corpus), `release-workflow` (SemVer + changelog + mcp↔plugin compatibility). Follow them rather than free-handing.
3. **Preserve compatibility.** New MCP tools are additive. Do not change existing tool signatures without a major bump and a plugin compatibility update. Keep the plugin's allowed-tool manifests in sync when adding tools.
4. **Test everything.** Per the dev conventions, new MCP capabilities ship with PHPUnit coverage. Each item below lists acceptance criteria — turn them into tests.
5. **Honesty over silence.** If a capability genuinely needs an external dependency (a Java validator, a terminology server), say so in the tool description and fail loudly with a clear message rather than returning plausible-but-fake output (e.g. fabricated MD5 hashes). This is the cardinal sin to avoid.

---

## Priority 0 — the "last mile" gaps that block real deliverables

### P0-1 · MCP tool: operational-template build (`template_build` / `opt_compile`)
- **Repo:** `openehr-assistant-mcp` (`src/Tools/`).
- **Problem (observed):** The session could author archetypes fine, but could **not** produce a real operational template. openEHR templates carry tool-computed `MD5-CAM-1.0.1` / `PARENT:MD5-CAM-1.0.1` checksums, a `buildUid`, and a full operational definition tree — none of which can be hand-fabricated (repo policy forbids it). The deliverable template had to be a deliberately incomplete "simplified" stub. There is no tool to turn an archetype set + template into a valid OPT.
- **Proposed change:** Add an MCP tool that accepts a root archetype id (or template definition) plus the set of referenced archetypes and returns a valid operational template (OPT / web-template JSON) with correctly computed `MD5-CAM` and `buildUid`. Realistic implementation paths, in order of preference:
  1. Shell out to a real openEHR compiler (Archie / `adl2-tools`) bundled or referenced as an external dependency.
  2. If no compiler is bundlable, implement deterministic **`MD5-CAM` computation** as a standalone tool (`template_md5` / `cam_hash`) — the CAM hash is an MD5 over canonicalised content and is computable in PHP — plus a `template_scaffold` tool that emits the correct structural skeleton.
- **Acceptance criteria:** Given the 6MWT archetype + `COMPOSITION.encounter.v1`, the tool emits a template whose `MD5-CAM` validates against an independent openEHR tool, and which a conformant renderer accepts. If a compiler is unavailable, the tool returns a structured error naming the missing dependency — never a fabricated hash.
- **Effort/risk:** High. This is the single highest-value gap. The compiler-bundling path is heavy; the `MD5-CAM`-computation path is the pragmatic MVP.

### P0-2 · MCP tool: authoritative ADL/AOM validation (`adl_validate`)
- **Repo:** `openehr-assistant-mcp` (`src/Tools/`).
- **Problem (observed):** `/archetype-lint` is an LLM applying the 22 normative *modelling* rules — strong on intent, but it is **not a parser**. Nothing in the session could give an authoritative machine parse to catch a misplaced brace, an invalid `C_DV_QUANTITY` interval, a malformed ordinal, or a broken path. Validation depth depended on the model reading carefully.
- **Proposed change:** Add a tool that runs a real ADL 1.4 / AOM parser (Archie-backed, or `adl-lt`) over an archetype and returns structured parse errors with line/column. Position it as complementary to `/archetype-lint`: the parser checks *syntax/structure*, the lint checks *semantics/modelling*.
- **Acceptance criteria:** Feeding a deliberately corrupted archetype (unbalanced `<>`, undefined at-code in `definition`) yields precise, located errors; a clean archetype yields a pass. The `/archetype-lint` skill is updated to call this tool first and only apply semantic rules to files that parse.
- **Effort/risk:** High (external parser dependency), but it removes a whole class of silent failure.

---

## Priority 1 — workflow friction that made reuse and authoring slower than they should be

### P1-1 · Plugin: give the authoring agent read-only MCP lookup access
- **Repo:** `openehr-assistant-plugin` (agent definition for `clinical-modeler`).
- **Problem (observed):** `clinical-modeler` can write local files but has **no MCP/CKM access**, while the agents that *do* have lookups (`ckm-scout`, `spec-researcher`) can't write. Every grounded fact (the property code `122`, the inline value-set idiom, the reuse list) had to be resolved in the main session and hand-carried into the implementer's prompt. This context-shuffling tax recurs on every authoring task.
- **Proposed change:** Grant `clinical-modeler` a **read-only** subset of MCP tools — `terminology_resolve`, `type_specification_get`, `guide_get`, `guide_adl_idiom_lookup`, `ckm_archetype_get` — so it can self-serve facts while authoring. Keep write/commit local. Alternatively, introduce a dedicated `modeller` agent that combines lookup + write.
- **Acceptance criteria:** A single dispatch to the authoring agent can resolve a property code and write a correct `C_DV_QUANTITY` without the controller pre-fetching it. No regression in context isolation (the agent still doesn't pull the whole session history).
- **Effort/risk:** Low — an agent tool-grant change. Watch token budget on the agent.

### P1-2 · Plugin/MCP: a "materialise CKM archetype into the workspace" path
- **Repo:** primarily `openehr-assistant-plugin` (new command/skill), leaning on existing `ckm_archetype_get`.
- **Problem (observed):** The reuse survey *recommended* reusing `pulse_oximetry`, `inspired_oxygen`, `device`, etc., but realising that reuse (for the full ATS/ERS version) needs the published native `.adl` written into `local/` and wired into a slot. That last step is entirely manual; reuse stayed advisory.
- **Proposed change:** Add a `/ckm-import` command (or extend the authoring workflow) that takes a CKM archetype id, fetches its native ADL via `ckm_archetype_get`, writes it to `local/` with the correct filename, and optionally inserts a constrained slot reference into a target archetype/template.
- **Acceptance criteria:** `/ckm-import openEHR-EHR-CLUSTER.inspired_oxygen.v1` produces `local/openEHR-EHR-CLUSTER.inspired_oxygen.v1.adl` and reports the slot-include snippet to paste. Reuse becomes one action, not a copy-paste chore.
- **Effort/risk:** Low–Medium.

### P1-3 · Guides/idioms: document `DV_SCALE` for non-integer rating scales
- **Repo:** `openehr-assistant-mcp` (`resources/guides/` + the ADL idioms cheatsheet behind `guide_adl_idiom_lookup`).
- **Problem (observed):** The modified Borg CR10 scale has a canonical `0.5` rung. `DV_ORDINAL` requires integer values, forcing a lossy integer-only subset. openEHR RM 1.1.0 introduced **`DV_SCALE`** — an ordinal-like type whose `value` is Real — which models exactly this case, but nothing in the guides or idioms surfaced it, so the limitation went unaddressed.
- **Proposed change:** Add a guide/idiom note: *use `DV_SCALE` (RM ≥ 1.1.0) for rating scales with non-integer steps (Borg CR10 0.5, etc.); use `DV_ORDINAL` for integer-only scales.* Include the ADL constraint syntax for both, and a tooling-support caveat (DV_SCALE needs RM 1.1.0-aware tooling; not all ADL 1.4 toolchains accept it). Consider having `/archetype-lint` emit an INFO suggesting `DV_SCALE` when it detects an ordinal whose rubrics imply half-steps.
- **Acceptance criteria:** `guide_adl_idiom_lookup("ordinal scale")` returns the DV_ORDINAL-vs-DV_SCALE guidance with both syntaxes and the version caveat.
- **Effort/risk:** Low — content + one lint heuristic. High knowledge value.

---

## Priority 2 — polish and coverage

### P2-1 · MCP: terminology-binding suggester (SNOMED CT / LOINC)
- **Repo:** `openehr-assistant-mcp` (`src/Tools/`).
- **Problem (observed):** `terminology_resolve` covers *openEHR* terminology well, but there was no way to *propose* external bindings (e.g. a SNOMED concept for "walking frame", or a LOINC code for the 6MWT). Coded leaves stayed local-only; external binding was fully manual.
- **Proposed change:** Add a `terminology_binding_suggest` tool that, given a concept label, proposes candidate SNOMED CT / LOINC codes (via an external terminology service or a bundled subset). Clearly mark candidates as *suggestions requiring human confirmation* (rule E4: bindings must be semantic equivalences, not approximations).
- **Acceptance criteria:** Suggesting bindings for "wheelchair" returns plausible SNOMED candidates with codes and FSNs, flagged as unverified. Returns a clear "no terminology service configured" error if the dependency is absent.
- **Effort/risk:** Medium–High (external service dependency). Scope carefully; an offline curated subset may be the MVP.

### P2-2 · Examples corpus: add the patterns this session lacked
- **Repo:** `openehr-assistant-mcp` (`resources/examples/archetypes/`), via `example-authoring`.
- **Problem (observed):** `examples_search` had no inline `DV_ORDINAL` exemplar and no timed-test / `state`+`protocol` OBSERVATION pattern, so the ordinal syntax leaned on canonical knowledge rather than a retrieved example.
- **Proposed change:** Curate examples for: (a) an inline `DV_ORDINAL` (and `DV_SCALE`) leaf; (b) an OBSERVATION exercising event-level `state` and observation-level `protocol`; ideally (c) a timed functional-test pattern.
- **Acceptance criteria:** `examples_search("ordinal scale")` and `examples_search("observation state protocol")` each return a curated hit with metadata headers.
- **Effort/risk:** Low.

### P2-3 · Lint rule C2 nuance for `ITEM_TREE` containers
- **Repo:** `openehr-assistant-mcp` (`resources/guides/archetypes/structural-constraints` and/or the lint rule text) and the plugin `archetype-lint` skill.
- **Problem (observed):** The lint flagged `items` cardinality `{0..*}` as an INFO against rule C2 ("containers default to `1..*`"), even though this is the established CKM convention for `ITEM_TREE.items` (e.g. the published `ecg_result.v1`). The INFO is noise on idiomatic code.
- **Proposed change:** Refine the rule guidance so `ITEM_TREE.items {0..*}` is recognised as conventional and not flagged when at least one contained ELEMENT is mandatory; reserve the C2 finding for genuinely empty/optional containers.
- **Acceptance criteria:** Linting `ecg_result.v1` and the 6MWT archetype produces no C2 INFO for the data `ITEM_TREE`.
- **Effort/risk:** Low.

### P2-4 · Authoring skills: a documented UID-generation step
- **Repo:** `openehr-assistant-plugin` (archetype-/template-authoring skills).
- **Problem (observed):** New archetypes/templates need a fresh `uid`, but `uuidgen` was absent in the environment (worked around via `/proc/sys/kernel/random/uuid`). The authoring skills don't specify a portable way to generate one.
- **Proposed change:** Add a short, portable UID-generation note to the authoring skills (`uuidgen` → `/proc/sys/kernel/random/uuid` → `python -c 'import uuid;print(uuid.uuid4())'` fallback chain), and reinforce that `uid` may be freshly generated but `MD5-CAM`/`buildUid` must never be hand-fabricated.
- **Acceptance criteria:** The authoring skills contain the fallback chain and the fabricate-nothing warning.
- **Effort/risk:** Trivial.

---

## Suggested sequencing

1. **P1-1** and **P1-3** first — cheap, high leverage, no external dependencies.
2. **P0-1** via the `MD5-CAM`-computation MVP (defer the full compiler bundling).
3. **P0-2** and **P2-1** once an external dependency strategy (Archie/terminology service) is agreed.
4. **P1-2**, **P2-2**, **P2-3**, **P2-4** as steady polish.

Each shipped item: bump versions and reconcile mcp↔plugin compatibility per `openehr-assistant-dev:release-workflow`, and update the changelog.
