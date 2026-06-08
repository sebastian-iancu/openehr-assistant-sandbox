# Refactor brief — cross-archetype diff, CKM import hygiene, and review-pipeline improvements

**Audience:** an AI refactor/feature agent working on `openehr-assistant-mcp` (the MCP server) and `openehr-assistant-plugin` (the Cursor/Claude plugin).
**Source:** friction observed while reviewing `openEHR-EHR-COMPOSITION.health_summary.v1` and diffing it against the CKM `openEHR-EHR-COMPOSITION.report.v1` on 2026-06-09: `/archetype-review` (intent/scope + 22-rule lint), a three-phrasing CKM reuse survey, and a cross-archetype `/archetype-diff`. See companion retrospective `docs/blog/2026-06-09-reviewing-and-diffing-a-composition-with-openehr-assistant.md`.
**Relationship to prior brief:** complements `docs/improvements/2026-06-09-openehr-assistant-review-and-impact-improvements.md` (the anatomical_location session). Where an item overlaps, it is marked **[reconfirms …]** with the new evidence; do not file duplicates — merge.
**Status:** proposal for triage — confirm each item against the real repos before implementing.

---

## Ground rules for the implementing agent

1. **Confirm repo layout first.** Paths below are inferred from plugin skill descriptors and MCP tool JSON in the plugin cache. Read `AGENTS.md` / `CONTRIBUTING.md` in the actual repos before editing.
2. **Use dev skills if available.** `mcp-tool-authoring`, `guide-prompt-authoring`, `example-authoring`, `release-workflow` — follow established patterns for new tools, guides, and SemVer bumps.
3. **Additive changes preferred.** New MCP tool parameters and skill modes should default to today's behaviour; breaking signatures need a major bump + plugin compatibility update.
4. **Test with real artefacts.** Acceptance criteria reference `COMPOSITION.health_summary.v1` and `COMPOSITION.report.v1` because they were this session's test case — two published, multi-language COMPOSITION *containers* that share a skeleton but diverge structurally.
5. **Do not fabricate validation output.** If a tool needs Archie, a live CKM ADL endpoint, or a reverse-dependency index, fail loudly with a structured error naming the missing dependency.

---

## Session findings summary (input for prioritisation)

| Area | What worked | What failed or was manual |
|---|---|---|
| Intent/scope + lint | `/archetype-review` staging; `guide_get` rules/structural/anti-patterns; `type_specification_get(COMPOSITION)` BMM check | Pipeline treats lint PASS as "nothing to improve"; no advisory-remediation path for WARN/INFO |
| Terminology | `terminology_resolve("433")` → `event` (openEHR group) | n/a this session (only one binding present) |
| CKM reuse survey | Three phrasings surfaced the container family + `report` twin | `health_summary` not returned for its own name "health summary"; no `rm_class` filter |
| Cross-archetype diff | `semantic-diff-rubric` lens exposed `at0002` repurposing + path collision | `/archetype-diff` refuses different root concepts; version-bump verdict meaningless cross-archetype |
| CKM fetch for diff | `ckm_archetype_get(report.v1)` returned full ADL | 16 language blocks pulled when only `definition` + `en` needed; no projection |
| Import hygiene | Provenance reasoning (MD5 mirror) done by model | No `ckm_archetype_diff`; local-vs-CKM identity never verified; no MD5/provenance guard |

---

## Priority 0 — diff and import-hygiene gaps (this session's core friction)

### P0-1 · `/archetype-diff` — add a cross-archetype (sibling) comparison mode

- **Repo:** `openehr-assistant-plugin` (command + `commands/references/semantic-diff-rubric.md`).
- **Problem:** The command and rubric are built to classify a **version bump of one archetype** and explicitly *refuse two different root concepts* (rubric "Notes for the implementer", last bullet). Comparing two **sibling containers** (`health_summary` vs `report`) is a legitimate, frequent question; the skill had to be coerced past its guard rail, and its patch/minor/major + rule-G1 verdict is meaningless across distinct archetypes.
- **Proposed change:**
  - Add a `mode` (or auto-detect on differing root `concept`): `version` (today's behaviour) vs `sibling`.
  - In `sibling` mode, downgrade the refusal to a one-line confirmation and emit a **compatibility/divergence report** instead of a bump verdict:
    - Shared skeleton (identical nodes/paths/constraints).
    - **Repurposed at-codes** (same code, different RM type/meaning) — flagged as the top breaking item.
    - **Path collisions** (same path, different RM type → AQL-incompatible).
    - Additive fields present in only one side.
    - Translation-coverage delta.
  - Replace "Recommended bump" with "Relationship: same archetype (versions) | siblings (incompatible) | one specialises the other".
- **Acceptance criteria:** `/archetype-diff health_summary.v1 report.v1` reports a shared COMPOSITION/`category=event`/open-content/Extension skeleton, flags `at0002` as repurposed (CLUSTER slot ↔ `ELEMENT` "Report ID") with a path collision at `/context/other_context[at0001]/items[at0002]`, and lists `report`'s `at0005` "Status" as additive — without emitting a misleading patch/minor/major verdict.
- **Effort/risk:** Low–Medium (mostly rubric + command prose; no backend).

### P0-2 · MCP tool: `ckm_archetype_diff` — local vs CKM published revision

- **Repo:** `openehr-assistant-mcp`. **[reconfirms prior brief P1-4 — now with fresh evidence.]**
- **Problem:** Verifying whether `local/health_summary.v1.adl` is byte-identical to its published CKM revision required a manual `ckm_archetype_get` + LLM diff, and in practice **was never closed** — the diff was run against the *twin* (`report`), not against CKM's own `health_summary`. There is no first-class import-hygiene check.
- **Proposed change:** `ckm_archetype_diff(local_path, identifier)` → fetch CKM ADL via the existing API and return:
  - Fast path: `MD5-CAM` / `uid` / `revision` equality (identical-or-not in one call).
  - Detail path: structural diff that separates **metadata** changes (revision, contributors, translations) from **definition** changes (nodes, constraints, bindings).
- **Acceptance criteria:** `ckm_archetype_diff(local/openEHR-EHR-COMPOSITION.health_summary.v1.adl, openEHR-EHR-COMPOSITION.health_summary.v1)` reports identical-to-published or a metadata-vs-definition split; runs without ad-hoc `curl`.
- **Effort/risk:** Medium (depends on a reachable CKM ADL endpoint; fail loudly if absent).

### P0-3 · MCP tool: `archetype_digest` (and/or `ckm_archetype_get` projection)

- **Repo:** `openehr-assistant-mcp`. **[reconfirms prior brief P0-3 — now also motivated by diffing.]**
- **Problem:** `ckm_archetype_get(report.v1)` returned **16 language blocks** when the diff needed only the `definition` section and the English `term_definitions`. Both definitions were then parsed into node trees by hand. No compact structural view; no payload projection.
- **Proposed change (either or both):**
  1. `ckm_archetype_get` gains `section` (`all` | `definition` | `header`) and `language` (`all` | code) parameters, defaulting to `all` for back-compat.
  2. `archetype_digest(path | identifier)` returning: root RM type + id + revision/uid; node inventory (at-codes with RM type, occurrences/cardinality); slot includes (regex); `term_bindings` by code system; per-language translation completeness (% `*(en)` placeholders).
- **Acceptance criteria:** A definition-only fetch of `report.v1` excludes the 15 non-English `details`/`term_definitions` blocks; `archetype_digest(health_summary.v1)` lists 3 definition at-codes (`at0000`, `at0001`, `at0002`), 1 Extension slot, `category` binding `openehr::433`, and 5 languages — without loading the full ADL into context.
- **Effort/risk:** Low (projection) / Medium (digest).

### P0-4 · Provenance / MD5 guard for CKM-mirror files

- **Repo:** `openehr-assistant-mcp` (tool) + `openehr-assistant-plugin` (review/edit skills).
- **Problem:** Nothing flagged that `local/health_summary.v1.adl` is a *published-CKM mirror*, nor warned that editing the English typo or the broken `nb` translation would invalidate `MD5-CAM-1.0.1` and diverge the file from its custodian. The "don't silently fork canonical" call rested entirely on the model.
- **Proposed change:**
  - Lightweight `archetype_provenance(path)` → `{ is_ckm_mirror, matched_identifier, ckm_revision, md5_matches }` (can wrap P0-2).
  - `/archetype-review` and any edit-capable skill: before proposing content edits, run the provenance check and, if it is a published mirror, surface a standard caveat ("edit invalidates MD5; prefer upstream CKM contribution").
- **Acceptance criteria:** Reviewing `health_summary.v1` prints a provenance line (mirror of published revision 1.0.3) and the edit-caveat *before* any suggested text fix.
- **Effort/risk:** Low–Medium (depends on P0-2).

---

## Priority 1 — command/skill friction observed this session

### P1-1 · `ckm_archetype_search` — exact-name ranking + `rm_class` filter

- **Repo:** `openehr-assistant-mcp`.
- **Problem:** Searching `health summary` did not return `COMPOSITION.health_summary` in the top 15 (it surfaced `health_risk`, `substance_use_summary`, …). Finding structural siblings needed three phrasings (`summary`, `document`, `report`).
- **Proposed change:**
  1. Boost ranking when the query matches an archetype's concept/`at0000` term (exact or near-exact) so a name search returns that archetype first.
  2. Add an optional `rm_class` parameter (`COMPOSITION` | `OBSERVATION` | …) to scope the survey to true structural siblings in one call.
- **Acceptance criteria:** `ckm_archetype_search("health summary", rm_class="COMPOSITION")` returns `COMPOSITION.health_summary` in the top results and lists `report`, `transfer_summary`, `medication_list`, `encounter`.
- **Effort/risk:** Low–Medium.

### P1-2 · `/archetype-review` — advisory remediation on a PASS

- **Repo:** `openehr-assistant-plugin` (`archetype-review` command).
- **Problem:** The pipeline routes a lint PASS straight to the optional review packet (Stage 6), skipping the fix stages. But the request was *"spot issues and suggest improvements"*, and a PASS-with-advisories (real WARNING/INFO items — `category` design choice, broken `nb`, English typo) still warrants structured suggestions. PASS ≠ nothing to improve.
- **Proposed change:** Add a lightweight branch: on PASS with non-empty WARNING/INFO, produce an **advisory remediation** block (issue → suggested fix → which fixes touch paths/semantics/MD5) without invoking the ERROR-only fix-plan/patch machinery.
- **Acceptance criteria:** Reviewing `health_summary.v1` yields a PASS *and* a short advisory list (nb translation, English typo) each tagged with its MD5/provenance impact.
- **Effort/risk:** Low (command prose).

### P1-3 · Skill preamble to pre-load deferred MCP tool schemas

- **Repo:** `openehr-assistant-plugin` (review/diff skills) — *may be a harness concern; confirm scope.*
- **Problem:** Each MCP tool (`guide_get`, `ckm_archetype_search`, `ckm_archetype_get`, `type_specification_get`, `terminology_resolve`) was a *deferred* tool requiring a discovery/schema-load step before its first call — extra round-trips when entering a new capability.
- **Proposed change:** Where the harness supports it, declare the tools each skill needs up front (so they are resolvable before first call), or add a one-line "load these tool schemas first" preamble to the review/diff skills.
- **Acceptance criteria:** A fresh session entering `/archetype-review` or `/archetype-diff` can call the required MCP tools without a separate discovery round-trip per tool.
- **Effort/risk:** Low (if in plugin scope).

### P1-4 · Skill parameter-name accuracy (carry-over check)

- **Repo:** `openehr-assistant-plugin`. **[reconfirms prior brief P1-1.]**
- **Note:** This session used `keyword` / `identifier` / `name` correctly on first attempt (schemas were loaded before calling), so the earlier param-drift friction did not recur here. Keep the prior brief's fix (generate skill examples from live schemas), and treat this session as evidence that *resolving schemas before first call* removes the problem.
- **Effort/risk:** Low.

---

## Priority 2 — quality-of-life

### P2-1 · `/archetype-diff` — emit a path-compatibility table

- **Repo:** `openehr-assistant-plugin`.
- **Problem:** The most actionable diff output for engineers was the **path collision** at `/context/other_context[at0001]/items[at0002]` (AQL written for one container breaks on the other). Today this is prose only.
- **Proposed change:** In `sibling` mode (P0-1), add a table of paths present in both with a per-path "compatible / type-changed / removed" verdict.
- **Effort/risk:** Low.

### P2-2 · Reuse-survey guide: "containers vs content" heuristic

- **Repo:** `openehr-assistant-mcp` (`resources/guides/`) or the reuse/authoring skill.
- **Problem:** The key survey insight — separating COMPOSITION *containers* (true siblings) from same-word EVALUATION *content* (what goes inside) — was model-derived, not guided.
- **Proposed change:** Add a short guide note: when surveying "similar" archetypes, group by RM class and distinguish container-level siblings from ENTRY-level content that would be *slotted into* the reviewed archetype.
- **Effort/risk:** Low (docs).

---

## Suggested content-fix backlog for the reviewed archetype (not MCP scope)

These are **content fixes** for `health_summary.v1` itself — track in CKM, not in the MCP repo, and note each invalidates `MD5-CAM-1.0.1` if applied locally:

| ID | Issue | Suggested fix | Rule |
|---|---|---|---|
| HS-1 | `nb` `use`: orphan fragment "som ved diabetes eller gravidiet."; missing the "clinicians managing specific aspects" reader bullet | Restore the missing bullet; fix `gravidiet` → `graviditet` | E6 (translation accuracy) |
| HS-2 | `en` `use`: "generic **containter**" | "container" | B2 (editorial) |
| HS-3 | `en` `use`: "familiar with **the all of the** relevant aspects" | "familiar with all the relevant aspects" | B2 (editorial) |
| HS-4 | `category` fixed to `433\|event\|` in the archetype | *Design decision, not a defect* — confirm snapshot semantics are intended globally; if persistent/episodic use is needed, relax to the `composition_category` group and pin per template (major version per G1) | C3 / G1 |

---

## Implementation order (recommended)

1. **P0-1** — `/archetype-diff` sibling mode (immediate; rubric + prose, no backend).
2. **P1-2** — `/archetype-review` advisory remediation (immediate; command prose).
3. **P0-3** — `ckm_archetype_get` projection / `archetype_digest` (unblocks cheap diffs + large-archetype review; shared with prior brief).
4. **P0-2** — `ckm_archetype_diff` (import hygiene; shared with prior brief P1-4).
5. **P0-4** — provenance/MD5 guard (builds on P0-2).
6. **P1-1** — search ranking + `rm_class` filter.
7. **P2-x** — path-compatibility table, reuse-survey guide note.

---

## Acceptance test pack (use after implementing)

Run against this sandbox + CKM:

```text
Archetype A (local):  local/openEHR-EHR-COMPOSITION.health_summary.v1.adl  (revision 1.0.3, MD5 B2D87AA…, 5 languages)
Archetype B (CKM):    openEHR-EHR-COMPOSITION.report.v1                     (revision 1.2.4, MD5 A60D0B7…, 16 languages)
Shared skeleton:      COMPOSITION · category=openehr::433 (event) · EVENT_CONTEXT.other_context ITEM_TREE[at0001] · unconstrained content · open CLUSTER Extension slot
Key divergence:       at0002 repurposed — CLUSTER "Extension" slot (A) vs ELEMENT "Report ID" DV_TEXT (B); path collision at /context/other_context[at0001]/items[at0002]
```

Expected outcomes after P0/P1 delivery:

- `/archetype-diff` (sibling mode) → relationship = "siblings (incompatible)"; flags `at0002` repurposing + path collision; lists `at0005` "Status" as additive; **no** patch/minor/major verdict.
- `ckm_archetype_diff(health_summary.v1 local, CKM id)` → identical-to-published, or a metadata-vs-definition split.
- `ckm_archetype_get(report.v1, section=definition, language=en)` → definition + English terms only (no 15 extra language blocks).
- `archetype_digest(health_summary.v1)` → 3 at-codes, 1 slot, `category` binding, 5 languages.
- `ckm_archetype_search("health summary", rm_class="COMPOSITION")` → `health_summary` ranked top; `report` family listed.
- `/archetype-review health_summary.v1` → PASS + advisory list (HS-1..HS-3) + provenance/MD5 caveat.

---

*Companion blog:* `docs/blog/2026-06-09-reviewing-and-diffing-a-composition-with-openehr-assistant.md`
*Prior brief (anatomical_location session):* `docs/improvements/2026-06-09-openehr-assistant-review-and-impact-improvements.md`
