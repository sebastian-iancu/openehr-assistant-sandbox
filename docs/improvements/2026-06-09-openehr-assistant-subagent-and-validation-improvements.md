# Refactor brief — openEHR Assistant subagent grounding & validation improvements

**Audience:** a refactor AI agent (or maintainer) working on the
`openehr-assistant-mcp` server and the `openehr-assistant-plugin` Claude Code plugin.
**Source:** findings from a real authoring session (2026-06-09) that produced
`local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl` (a new 6MWT OBSERVATION,
STRICT-lint-clean) and `local/Six minute walk test.t.json` (a `report-result`-rooted
web template composing the 6MWT with reused pulse/blood_pressure/pulse_oximetry). See
the companion retrospective in
[`docs/blog/2026-06-09-modelling-a-six-minute-walk-test-archetype-and-template.md`](../blog/2026-06-09-modelling-a-six-minute-walk-test-archetype-and-template.md).

## How to use this brief

Each item has: **Repo · Problem · Evidence · Proposed fix · Acceptance test ·
Priority (P0–P2) · Effort**. Do not bundle unrelated changes. Verify each fix against
its acceptance test. Where a fix spans both repos, implement the MCP side first.

Repos referenced:
- **MCP** = `openehr-assistant-mcp` (server: `#[McpTool]`s, `resources/guides/`,
  `resources/examples/`, agent/permission config it ships).
- **PLUGIN** = `openehr-assistant-plugin` (skills, commands, agents).

**Relationship to prior briefs:** this session overlaps the
[template-authoring brief](2026-06-09-openehr-assistant-template-authoring-improvements.md)
(items 1/3/8 there: a template validator, a templates example namespace, the
OET→`.t.json` bridge) and the
[tooling brief](2026-06-09-openehr-assistant-tooling-improvements.md). Where this
session adds **new evidence or a new dimension** (the `.t.json`/CAM format, subagent
tool access, property-code grounding) it is called out below; pure duplicates are noted
as *reinforces* rather than restated.

---

## P0 — grounding reach (blocks the agent that's built for this)

### 1. Give the `clinical-modeler` agent real access to its MCP grounding tools
- **Repo:** PLUGIN (agent definition + permission inheritance) — possibly MCP/host config.
- **Problem:** The `clinical-modeler` agent's definition lists
  `terminology_resolve`, `ckm_archetype_get`, `guide_get`, `guide_adl_idiom_lookup`,
  `type_specification_get` — but at runtime those MCP tools were **not callable** from
  the subagent. The one agent purpose-built for *grounded* authoring had to fall back to
  guessing openEHR property codes from on-disk archetype copies.
- **Evidence:** Both implementer subagents reported MCP unavailable
  ("`terminology_resolve` and `ckm_archetype_get` unavailable… not pre-approved for the
  subagent"). The archetype implementer inferred `property=[openehr::380/382/122]` from
  disk; the main session (which *does* have MCP) then had to verify them. The grounding
  the agent was designed around never reached it.
- **Proposed fix:** Ensure the MCP tool permissions that the plugin requests also apply
  to its spawned subagents — e.g. the shipped allow-list (see item 3) must cover the
  subagent context, or the agent frontmatter must declare the MCP tools in a way the
  host actually grants. Add a smoke test: a `clinical-modeler` dispatch that calls
  `terminology_resolve("Length")` and returns `122`.
- **Acceptance test:** A `clinical-modeler` subagent can successfully call
  `terminology_resolve` and `ckm_archetype_get` without the main session relaying results.
- **Priority:** P0 · **Effort:** S–M

### 2. Web-template (`.t.json` / OPT / CAM) validator + a content-bearing example
- **Repo:** MCP (+ PLUGIN command wrapper). *Extends* template-authoring-brief items 1 & 3
  to the canonical JSON/CAM format (those were framed around OET).
- **Problem:** There is no way to validate a `.t.json` web template — neither its
  `C_ARCHETYPE_ROOT` `content` tree nor its narrowing. And the repo's only COMPOSITION
  template, `Vital signs.t.json`, is an **empty `encounter` shell** (context only, no
  content), so there is **no worked example** of the exact pattern needed: a COMPOSITION
  root filling `content` with entry archetypes.
- **Evidence:** This session hand-wrote the `content` `C_ATTRIBUTE` →
  `C_ARCHETYPE_ROOT[×4]` JSON skeleton into the implementer's prompt because no example
  existed and `examples_search` has no `templates` namespace (and isn't reachable from
  `clinical-modeler` — see item 1). Validation was done with throwaway Python
  (`json.tool` + a structural walk of `definition.attributes`).
- **Proposed fix:** (a) A `template_validate(file, {format: oet|opt|tjson})` tool that
  resolves each content `C_ARCHETYPE_ROOT`/`Rule path` against the referenced archetype
  and checks legal narrowing, returning an `archetype-lint`-style report; (b) seed
  `openehr://examples/templates/` with **at least one content-bearing `.t.json`** (e.g.
  this session's `Six minute walk test.t.json`) carrying a metadata header that
  documents the `C_ARCHETYPE_ROOT` shape and occurrence overrides.
- **Acceptance test:** `template_validate("local/Six minute walk test.t.json")` returns
  0 errors; flipping a content `archetypeId` to a non-existent id yields an ERROR;
  `examples_search("composition content observation template")` returns the new example.
- **Priority:** P0 · **Effort:** M–L

---

## P1 — reliability & onboarding friction

### 3. Ship (or document) the MCP permission allow-list for the content sandbox
- **Repo:** PLUGIN / MCP (+ docs).
- **Problem:** A fresh content workspace had **no MCP permission allow-list**, so the
  first `mcp__…openehr-assistant` calls failed ("Tool permission request failed").
  Work only proceeded after manually creating `.claude/settings.json` with
  `permissions.allow: ["mcp__openehr-assistant", "mcp__plugin_openehr-assistant_openehr-assistant"]`.
- **Evidence:** The opening CKM survey and the `AskUserQuestion` both failed on
  permission errors until the allow-list was added mid-session. The plugin repo already
  ships its own `.claude/settings.json`; the companion *content* repo did not.
- **Proposed fix:** Either ship a starter `.claude/settings.json` in the content-repo
  template, or have the plugin advertise the required permission scopes in its README /
  `SessionStart` hook so a new workspace is usable without trial-and-error. Note that
  both tool prefixes are needed (`mcp__openehr-assistant` and the
  `mcp__plugin_openehr-assistant_openehr-assistant` plugin-namespaced form).
- **Acceptance test:** A brand-new clone of the content repo can call an MCP tool on the
  first turn without a manual settings edit.
- **Priority:** P1 · **Effort:** XS–S

### 4. Fix `ckm-scout` agent reliability (stalls with zero tool calls)
- **Repo:** PLUGIN (`ckm-scout` agent).
- **Problem:** The dedicated reuse-survey agent **narrated its search plan repeatedly
  but never issued a tool call**, returning with `tool_uses: 0` — twice. A reuse-first
  specialist that doesn't run `ckm_archetype_search` is worse than calling the tool
  directly.
- **Evidence:** Two `ckm-scout` dispatches this session ended at 0 tool uses (one after
  a permission rejection, one after ~7s of pure narration). The survey ultimately ran by
  calling `ckm_archetype_search` directly from the main session.
- **Proposed fix:** Tighten the `ckm-scout` system prompt to *issue the parallel
  searches immediately* (lead with the tool calls, ban pre-amble narration), and ensure
  its tool permissions are granted in the subagent context (see item 1). Add a guard/eval
  that a `ckm-scout` run performs ≥1 `ckm_archetype_search`.
- **Acceptance test:** A `ckm-scout` dispatch for "six minute walk test" returns a ranked
  shortlist and reports ≥3 tool calls.
- **Priority:** P1 · **Effort:** S

### 5. Add a `DV_QUANTITY` units→property idiom table to the ADL idioms cheatsheet
- **Repo:** MCP (`resources/guides/archetypes/adl-idioms-cheatsheet`).
- **Problem:** Choosing the right `property=[openehr::NNN]` for a `DV_QUANTITY` unit is a
  manual leap that produced **two independent wrong guesses** this session.
- **Evidence:** The plan suggested `125` (resolves to *Pressure*) for a `%` value and
  `384` (*Amount of substance*) for `/min`; the subagent inferred different codes from
  disk; only `terminology_resolve` settled the correct trio: `122` Length (m), `382`
  Frequency (/min), `380` Qualified real (%).
- **Proposed fix:** Add a small reference table of common clinical units and their
  canonical openEHR property codes (length=122/`m`, mass=124/`kg`, pressure=125/`mm[Hg]`,
  temperature=127, frequency=382/`/min`, qualified real=380/`%`, …), with a one-line
  pointer to verify via `terminology_resolve(groupId="property")`.
- **Acceptance test:** A reader can pick the property for "heart rate in /min" from the
  table without resolving codes one by one.
- **Priority:** P1 · **Effort:** XS

---

## P2 — depth & correctness of validation

### 6. Add an Archie-backed `archetype_parse` / true-syntax validator
- **Repo:** MCP.
- **Problem:** `archetype-lint` is rule-based heuristics, **not a parser**. It cannot
  confirm true ADL 1.4 syntactic validity, so genuine grammar questions are left to
  human/agent judgement.
- **Evidence:** This session had open syntax micro-decisions the linter can't
  adjudicate — `DV_BOOLEAN` `{true,false}` vs `{True,False}` (the implementer changed it
  on a corpus hunch), and the `DV_ORDINAL` `n|[local::atNNNN]` list / `C_DV_QUANTITY`
  block were validated only by eye and by the rule engine.
- **Proposed fix:** Expose a real parse/validation (Archie or `adl-lt`) as
  `archetype_parse(file)` returning AOM validation codes (the `VATDF`/`VARDT`/… family
  already referenced in the rules guide). Keep `archetype-lint` as the *design*-rules
  layer on top of a confirmed parse.
- **Acceptance test:** `archetype_parse` accepts the 6MWT archetype and reports a precise
  error when `DV_BOOLEAN` is given an illegal literal.
- **Priority:** P2 · **Effort:** L

### 7. Guide note: non-integer rating scales → `DV_SCALE` and RM-version implications
- **Repo:** MCP (`archetypes/structural-constraints` / idioms cheatsheet §5).
- **Problem:** Modelling the modified Borg CR10 scale exposed that its 0.5 rung **cannot**
  be represented by an integer `DV_ORDINAL`, and `DV_SCALE` requires RM ≥ 1.1.0 — a
  trade-off that should be surfaced at design time, not discovered mid-build.
- **Evidence:** This session deliberately dropped the Borg 0.5 step under ADL 1.4 (the
  cheatsheet mentions `DV_SCALE` exists but not the version/representability consequence).
- **Proposed fix:** Add a short note: "rating scales with non-integer rungs (e.g.
  modified Borg 0.5) need `DV_SCALE` (RM ≥ 1.1.0); under ADL 1.4 / RM 1.0.x they must be
  approximated by integer `DV_ORDINAL`, dropping fractional rungs."
- **Acceptance test:** A reader choosing an ordinal-vs-scale data type sees the
  representability/version caveat before authoring.
- **Priority:** P2 · **Effort:** XS

### 8. Make `examples_search` reach templates — and reach the `clinical-modeler` agent
- **Repo:** MCP (+ PLUGIN agent toolset). *Reinforces* template-authoring-brief item 3,
  with a second concrete miss (the CAM `.t.json` format).
- **Problem:** `examples_search` covers `aql | flat | structured | archetypes` only, and
  is not in `clinical-modeler`'s toolset — so a template-authoring subagent cannot pull a
  worked template example even if one existed.
- **Proposed fix:** Add the `templates` namespace (item 2b) **and** add `examples_search`/
  `examples_get` to the `clinical-modeler` agent's allowed tools so it can self-serve the
  pattern instead of needing the skeleton injected by the dispatcher.
- **Acceptance test:** A `clinical-modeler` dispatch can `examples_get` a content-bearing
  template example without the main session providing it.
- **Priority:** P2 · **Effort:** S

---

## Summary matrix

| # | Title | Repo | Priority | Effort |
|---|-------|------|----------|--------|
| 1 | `clinical-modeler` subagent gets real MCP grounding access | PLUGIN | P0 | S–M |
| 2 | `.t.json`/OPT `template_validate` + content-bearing example | MCP | P0 | M–L |
| 3 | Ship/document MCP permission allow-list for content sandbox | PLUGIN/MCP | P1 | XS–S |
| 4 | Fix `ckm-scout` stalling (0 tool calls) | PLUGIN | P1 | S |
| 5 | `DV_QUANTITY` units→property idiom table | MCP | P1 | XS |
| 6 | Archie-backed `archetype_parse` (true ADL validation) | MCP | P2 | L |
| 7 | Guide note: non-integer scales → `DV_SCALE` + RM version | MCP | P2 | XS |
| 8 | `examples_search` templates namespace + give to clinical-modeler | MCP/PLUGIN | P2 | S |

**Cross-cutting theme:** the knowledge layer (guides, CKM search, terminology) is
strong; the gaps are **verification depth** (real parse, template validation) and
**grounding reach** (the authoring subagent and `ckm-scout` couldn't actually call the
tools they're defined with). Fixing item 1 unblocks the agents this plugin's whole
authoring story depends on.
