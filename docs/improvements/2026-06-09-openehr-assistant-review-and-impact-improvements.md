# Refactor brief — review, rationale, and impact-analysis improvements

**Audience:** an AI refactor/feature agent working on `openehr-assistant-mcp` (the MCP server) and `openehr-assistant-plugin` (the Cursor/Claude plugin).
**Source:** friction observed while reviewing `openEHR-EHR-CLUSTER.anatomical_location.v1` end-to-end on 2026-06-09: lint, rationale drafting, semantic explain, CKM family survey, and workspace impact analysis. See companion retrospective `docs/blog/2026-06-09-reviewing-anatomical-location-with-openehr-assistant.md`.
**Status:** proposal for triage — confirm each item against the real repos before implementing.

---

## Ground rules for the implementing agent

1. **Confirm repo layout first.** Paths below are inferred from plugin skill descriptors and MCP tool JSON in the Cursor plugin cache. Read `AGENTS.md` / `CONTRIBUTING.md` in the actual `openehr-assistant-mcp` and `openehr-assistant-plugin` repos before editing.
2. **Use dev skills if available.** `mcp-tool-authoring`, `guide-prompt-authoring`, `example-authoring`, `release-workflow` — follow established patterns for new tools, guides, and SemVer bumps.
3. **Additive changes preferred.** New MCP tools and guide content should not break existing tool signatures without a major version bump and plugin compatibility update.
4. **Test with real artefacts.** Acceptance criteria reference `CLUSTER.anatomical_location.v1` because it was the session's test case — a large (2 142-line), published, multi-language archetype with SNOMED bindings and slot consumers.
5. **Do not fabricate validation output.** If a tool needs Archie, a SNOMED API, or CKM reverse-dependency endpoints, fail loudly with a structured error naming the missing dependency.

---

## Session findings summary (input for prioritisation)

| Area | What worked | What failed or was manual |
|---|---|---|
| Lint / quality review | `guide_get(archetypes/rules)` + checklist; 22-rule mental model | No `archetype_lint` MCP tool; 2 142-line file chunked by hand |
| Rationale prose | `language-standards`, sibling ADL from `ckm_archetype_get` | SNOMED bindings not resolvable via `terminology_resolve` |
| CKM survey | `ckm_archetype_search` found location family with status/revision | First call used wrong param `query` instead of `keyword` |
| RM clarity | `type_specification_get(CLUSTER)` after fixing param `name` | First call used wrong param `type_name` |
| Impact analysis | Found `Health Certificate.t.json` + 3 parent `.adl` slot constraints | Command glob omitted `.t.json`; no CKM reverse-deps |
| Archetype defects found | Catalan `at0067` label wrong; `clock` vs `circle` in use text | No automated diff vs CKM published ADL |

---

## Priority 0 — determinism gaps that weaken review commands

### P0-1 · MCP tool: `archetype_lint`

- **Repo:** `openehr-assistant-mcp` (`src/Tools/` or equivalent).
- **Problem:** `/archetype-lint` is a skill instructing an LLM to apply 22 rules from `guide_get(archetypes/rules)`. Output varies by model and context. The anatomical_location review could not produce reproducible violations JSON.
- **Proposed change:** Add `archetype_lint` MCP tool:
  - **Input:** `path` (local file) OR `identifier` (CKM id → fetch ADL), `mode` (`STRICT` | `PERMISSIVE`, default `PERMISSIVE`).
  - **Output:** JSON matching the skill's report schema (rule id, severity, explanation, suggested fix); overall PASS/FAIL.
  - **Implementation:** Phase 1 — rule checks implementable without a full parser (at-code completeness, RM root type match, slot regex presence, term_definitions keys). Phase 2 — integrate Archie/adl-lt for syntax invariants (VARID, VARDF, VATDF, …) before semantic rules.
- **Acceptance criteria:**
  - `archetype_lint(local/openEHR-EHR-CLUSTER.anatomical_location.v1.adl)` returns PASS in PERMISSIVE with documented WARNINGs for translation placeholders (if rules cover i18n).
  - Same input → same violations list across runs (deterministic Phase 1 rules).
  - `/archetype-lint` skill updated to call this tool first, then add interpretive prose only for WARNING justification.
- **Effort/risk:** High (Phase 2); Medium (Phase 1 only).

### P0-2 · MCP tool: `snomed_resolve` (or extend `terminology_resolve`)

- **Repo:** `openehr-assistant-mcp`.
- **Problem:** `/archetype-rationale` requires grounding in terminology bindings. `anatomical_location.v1` binds laterality to SNOMED CT codes. `terminology_resolve` only searches openEHR terminology — all four SNOMED codes returned "Could not resolve."
- **Proposed change:** Either:
  1. Extend `terminology_resolve` with `system` parameter (`openehr` | `SNOMED-CT` | `LOINC` | …), or
  2. Add dedicated `snomed_resolve(code)` returning preferred term + semantic tag + active status.
- **Acceptance criteria:**
  - `snomed_resolve("7771000")` → Left (qualifier value) or equivalent SNOMED preferred term.
  - `/archetype-rationale` skill updated to call this for `term_bindings` entries before drafting prose.
  - Tool description states licensing/affiliate requirements for SNOMED CT (archetype already carries IHTSDO acknowledgement).
- **Effort/risk:** Medium (depends on SNOMED API/licensing). If no API available, document limitation in command and return `TODO: SNOMED lookup unavailable`.

### P0-3 · MCP tool: `archetype_digest`

- **Repo:** `openehr-assistant-mcp`.
- **Problem:** Published CKM archetypes routinely exceed 100 KB. Review required chunked `Read`, manual `grep`, and a one-off Python script for at-code completeness. No compact structural summary.
- **Proposed change:** `archetype_digest(path | identifier)` returning:
  - Root RM type, archetype id, revision/uid from header
  - Node inventory: at-codes in `definition` with occurrences/cardinality
  - Slot includes (regex patterns)
  - `term_bindings` summary by code system
  - Per-language translation completeness (% terms with `*(en)` placeholders)
  - Line count / section offsets for chunked reads
- **Acceptance criteria:** Digest of `anatomical_location.v1` lists 35 definition at-codes, 2 slots, 4 SNOMED bindings, 11 translation languages — without loading full ADL into the LLM context.
- **Effort/risk:** Medium.

---

## Priority 1 — command/skill friction observed this session

### P1-1 · Align skill examples with live MCP tool schemas

- **Repo:** `openehr-assistant-plugin` (skills under `skills/`).
- **Problem:** Repeated first-call failures:
  - `ckm_archetype_search({ query })` → requires `keyword`
  - `ckm_archetype_get({ cid })` → requires `identifier`
  - `type_specification_get({ type_name })` → requires `name`
- **Proposed change:**
  1. Generate skill `allowed-tools` invocation examples from MCP tool JSON descriptors at build time, OR
  2. Add a `mcp_tool_schema` meta-tool that returns parameter names for a given tool before first use.
  3. Update `archetype-authoring`, `archetype-lint`, `/archetype-rationale`, `/archetype-explain` skills with correct parameter names.
- **Acceptance criteria:** Fresh agent session calls `ckm_archetype_search` successfully on first attempt without reading descriptor files manually.
- **Effort/risk:** Low–Medium.

### P1-2 · Extend `/archetype-impact` for this repo's artefact types

- **Repo:** `openehr-assistant-plugin` (command definition).
- **Problem:** Command globs `*.oet`, `*.opt`, `*.aql`, `*.sql`, `*.md` but this sandbox (and many modern workflows) uses `*.t.json` web templates. Impact on `anatomical_location.v1` was found only via broader grep.
- **Proposed change:** Update command instructions:
  ```
  Glob: **/*.t.json
  Glob: **/*.adl   # parent archetype slot constraints
  ```
  Add output section **Parent archetypes (slot constraints)** with grep pattern for `anatomical_location` regex in `allow_archetype` blocks.
- **Acceptance criteria:** `/archetype-impact anatomical_location.v1` on this sandbox reports `Health Certificate.t.json` and three parent `.adl` files without ad-hoc search.
- **Effort/risk:** Low.

### P1-3 · MCP tool: `archetype_impact` (optional automation of P1-2)

- **Repo:** `openehr-assistant-mcp`.
- **Proposed change:** `archetype_impact(identifier, workspace_root)` → structured JSON of templates, parent archetype slots, aql/md mentions.
- **Acceptance criteria:** Same results as manual grep for anatomical_location test case.
- **Effort/risk:** Low–Medium.

### P1-4 · MCP tool: `ckm_archetype_diff`

- **Repo:** `openehr-assistant-mcp`.
- **Problem:** Could not verify local `anatomical_location.v1.adl` against CKM revision 1.5.0. `curl` to CKM ADL endpoint failed in sandbox; no first-class diff tool.
- **Proposed change:** `ckm_archetype_diff(local_path, identifier)` → fetch CKM ADL via existing API, return unified diff or hash comparison (`MD5-CAM`, uid, revision).
- **Acceptance criteria:** Reports whether local file matches published revision; highlights metadata vs definition changes separately.
- **Effort/risk:** Medium.

### P1-5 · `/archetype-rationale` — document SNOMED limitation and sibling list

- **Repo:** `openehr-assistant-plugin`.
- **Problem:** Command step 4 mandates `terminology_resolve` for bound codes; fails silently for SNOMED. Sibling fetch (`device.v1`) was auto-blocked; `anatomical_location_relative.v2` was the workable substitute but not documented in the command.
- **Proposed change:** Update command instructions:
  - "For SNOMED/LOINC bindings, use `snomed_resolve` (P0-2) or read rubrics from archetype `term_definitions`."
  - "Preferred prose-style siblings for CLUSTER location family: `anatomical_location_relative.v2`, `anatomical_location_circle.v1`."
- **Effort/risk:** Low (docs only until P0-2 ships).

---

## Priority 2 — quality-of-life and CKM-scale features

### P2-1 · CKM reverse-dependency search for impact

- **Repo:** `openehr-assistant-mcp` (new tool or CKM API integration).
- **Problem:** Local impact found 1 template + 3 parents; CKM production impact is orders of magnitude larger. `/archetype-impact` correctly disclaims CKM scope but users will ask anyway.
- **Proposed change:** `ckm_archetype_dependents(identifier)` → archetypes/templates referencing this id (if CKM API supports it) or indexed mirror.
- **Acceptance criteria:** Returns a bounded list (top N) with cid, archetypeId, reference type (slot/include).
- **Effort/risk:** High (depends on CKM API availability).

### P2-2 · Translation lint rules in `archetype_lint`

- **Repo:** `openehr-assistant-mcp` + `resources/guides/archetypes/rules`.
- **Problem:** Session found Catalan `at0067` = "Línia mitjana" (should be Medial); many `*(en)` placeholders in `ar-sy`/`sl`. No rule ID covered translation accuracy.
- **Proposed change:** Add rule **R23 Translation accuracy** (WARNING): flag `*(en)` stubs, identical wrong labels across at-codes in non-English blocks, copy-paste errors (Medial vs Midline).
- **Acceptance criteria:** `archetype_lint` flags Catalan `at0067` in anatomical_location.v1.
- **Effort/risk:** Medium (heuristic; may need human-review flag).

### P2-3 · Metadata consistency rule: prose vs slot constraints

- **Repo:** guides + lint tool.
- **Problem:** `use` text references `CLUSTER.anatomical_location_clock`; slot allows `anatomical_location_circle`. LLM review caught it; no machine check.
- **Proposed change:** Lint INFO/WARNING when `use`/`misuse`/`comment` mention archetype ids not matching slot `include` regexes in `definition`.
- **Acceptance criteria:** Flags `anatomical_location_clock` vs `circle` mismatch in anatomical_location.v1.
- **Effort/risk:** Low–Medium.

### P2-4 · `examples_get` for review workflows

- **Repo:** plugin command or skill update.
- **Problem:** `examples_search` confirmed gold-standard match by UID snippet but review workflow does not mandate loading the example for diff.
- **Proposed change:** `/archetype-lint` optional step: if examples corpus contains same archetype id, `examples_get` and diff uid/revision/hash.
- **Effort/risk:** Low.

### P2-5 · Auto-review policy for read-only sibling fetches

- **Repo:** Cursor plugin / agent policy (may be outside openehr-assistant repos).
- **Problem:** `ckm_archetype_get(device.v1)` blocked as scope-widening during `/archetype-rationale` even though read-only and intended for prose style.
- **Proposed change:** Document allowlist of read-only CKM fetches for rationale command; or fetch by RM type + "published CLUSTER in Common resources" query instead of hard-coded sibling id.
- **Effort/risk:** Low (policy/docs) if not configurable in plugin.

---

## Suggested fix backlog for the reviewed archetype (not MCP scope)

These are **content fixes** for `anatomical_location.v1` itself — track in CKM, not in the MCP repo:

| ID | Issue | Suggested fix | Rule |
|---|---|---|---|
| AL-1 | Catalan `at0067` text "Línia mitjana" | Change to Medial equivalent; keep description | Translation accuracy |
| AL-2 | `use` text says `anatomical_location_clock` | Align to `anatomical_location_circle` per slot L394 | Editorial consistency (P2-3) |
| AL-3 | `ar-sy` / `sl` term_definitions mostly `*(en)` | Complete translations or mark languages draft | R8 / translation guide |
| AL-4 | Partial SNOMED binding (laterality only) | Optional: bind aspect value set where semantically equivalent | R18 INFO |

---

## Implementation order (recommended)

1. **P1-1** — schema-aligned skill examples (immediate friction reduction, no new backend).
2. **P1-2** — `/archetype-impact` glob update (immediate, docs-only).
3. **P0-3** — `archetype_digest` (unblocks all large-archetype review).
4. **P0-1** — `archetype_lint` Phase 1 (deterministic core).
5. **P0-2** — `snomed_resolve` (unblocks rationale grounding).
6. **P1-4** — `ckm_archetype_diff` (import hygiene).
7. **P0-1 Phase 2** + **P2-1** — parser + CKM dependents (longer horizon).

---

## Acceptance test pack (use after implementing)

Run against this sandbox:

```text
Archetype:  local/openEHR-EHR-CLUSTER.anatomical_location.v1.adl
CKM id:     openEHR-EHR-CLUSTER.anatomical_location.v1 (1013.1.587)
Template:   Health Certificate.t.json (overlay parentArchetypeId)
Parents:    exam.v2 (at0011), problem_diagnosis.v1 (at0039), imaging_exam_result.v1 (at0006)
SNOMED:     272741003, 7771000, 24028007, 51440002
```

Expected outcomes after full P0/P1 delivery:

- `archetype_lint` → PERMISSIVE PASS, WARNINGs for AL-1–AL-3.
- `archetype_digest` → 35 nodes, 2 slots, 11 languages, ~2142 lines.
- `snomed_resolve("7771000")` → non-empty preferred term.
- `archetype_impact` → 1 template, 3 parent archetypes.
- `ckm_archetype_diff` → identical or unified diff vs revision 1.5.0.

---

*Companion blog:* `docs/blog/2026-06-09-reviewing-anatomical-location-with-openehr-assistant.md`
