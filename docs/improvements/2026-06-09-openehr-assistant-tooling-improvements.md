# Refactor brief — openEHR Assistant tooling improvements

**Date:** 2026-06-09
**Status:** Proposed — for a refactor/maintenance AI agent
**Origin:** Design-review + QA session on [openEHR-EHR-COMPOSITION.health_summary.v1.adl](../../local/openEHR-EHR-COMPOSITION.health_summary.v1.adl)
**Targets:** `openehr-assistant-mcp` (the MCP server) and `openehr-assistant-plugin` (commands + skills + guides)
**Companion:** [docs/blog/2026-06-09-reviewing-a-published-health-summary-composition.md](../blog/2026-06-09-reviewing-a-published-health-summary-composition.md)

## How to use this file

Each item is an independent, actionable change request. Fields: **Target** (which repo), **Type** (new tool / guide edit / skill edit / data), **Priority** (P1 high → P3 nice-to-have), **Problem**, **Evidence** (what in the session exposed it), **Proposed change**, **Acceptance criteria**. Work P1s first; they remove the most manual reasoning from a review. Do **not** change archetype content in the sandbox as part of this brief — these are *tooling* improvements.

---

### IMP-1 — Add a real ADL/AOM validator tool (P1)

- **Target:** `openehr-assistant-mcp`
- **Type:** new MCP tool
- **Problem:** The "22 normative lint rules" and the tooling validity codes (`VARID, VARCN, VARDF, VARON, VARDT, VATDF, VACDF, VDFAI, VDFPT`) are applied by the model reading guide prose and eyeballing the ADL. There is no deterministic check. At-code parity across locales, slot validity, RM attribute-name correctness, and cardinality/occurrences/existence invariants are all confirmed by hand — error-prone and non-reproducible.
- **Evidence:** In the health_summary review, at-code parity across en/nb/es/pt-br/sv and the `category` slot validity were verified manually; the only RM grounding available was `type_specification_get`, which returns the model but does not validate the instance against it.
- **Proposed change:** Expose `archetype_validate(identifier_or_adl, mode=strict|permissive)` backed by an AOM validator (e.g. `archie`). Return a structured report: per-rule pass/fail, the emitted `VAR*`/`VATDF`/`VACDF` codes, and AOM 1.4 structural-invariant results. Have the `archetype-lint` skill call this tool instead of (or before) prose-based reasoning.
- **Acceptance:** Running the tool on a known-good CKM archetype yields all-pass; on a file with an undefined at-code or a bad RM attribute name it returns the specific failing code and node path.

---

### IMP-2 — Provenance / CKM-mirror detection tool (P1)

- **Target:** `openehr-assistant-mcp`
- **Type:** new MCP tool
- **Problem:** The most decisive review insight — "this local file is an unmodified mirror of a published CKM artefact" — required manually fetching CKM and comparing `uid` + `MD5-CAM` by eye. This safety check (don't silently fork published, CC-BY-SA content) should be one call.
- **Evidence:** The health_summary file matched the published artefact exactly (`uid aab23408-…abbd`, `MD5-CAM B2D87AA5…44F7`, revision 1.0.3, `lifecycle_state published`). Knowing this *reframed every finding* from "local fix" to "upstream CR".
- **Proposed change:** `archetype_provenance(local_file)` → `{ is_published_mirror: bool, ckm_identifier, ckm_revision, lifecycle_state, divergence: [...] }`. It should look up the `uid`/identifier in CKM and report whether the local file matches, and if not, summarise divergence. Surface the result as the first output of the `review-remediate` pipeline's *intent & provenance* stage.
- **Acceptance:** On a verbatim CKM mirror it returns `is_published_mirror: true` with matching revision and an empty divergence list; on a locally-edited copy it lists the changed sections.

---

### IMP-3 — Checksum recompute / verify tool (P1)

- **Target:** `openehr-assistant-mcp`
- **Type:** new MCP tool
- **Problem:** When a file carries `MD5-CAM-1.0.1` and `build_uid`, the reviewer cannot confirm whether the checksum is *valid* or *stale* — there is no recompute capability. Claims about checksum validity are reasoned, not verified.
- **Evidence:** I had to hedge: "the checksum *still matches* because content is semantically identical" — an inference I could not verify.
- **Proposed change:** `archetype_checksum(identifier_or_adl)` that recomputes the CAM checksum over the canonical model form and reports `{ stored, computed, matches }`. Pair it with IMP-2 so provenance + checksum validity report together.
- **Acceptance:** Recomputing on an untouched CKM file reproduces the stored `MD5-CAM`; after a content edit, `matches: false`.

---

### IMP-4 — Wire semantic-diff (local ↔ CKM) into the review pipeline (P2)

- **Target:** `openehr-assistant-plugin` (skill `archetype-authoring` / `review-remediate.md`), optionally a thin MCP helper
- **Type:** skill edit + (optional) tool
- **Problem:** The `semantic-diff` command exists but is not invoked from the review flow, and it doesn't report checksum validity. Drift between a workspace file and its CKM origin is computed in the model's head.
- **Evidence:** I diffed the workspace ADL against `ckm_archetype_get` output manually, noticing only cosmetic field-order drift by reading both blobs.
- **Proposed change:** In `review-remediate.md`, make the provenance stage call `semantic-diff(local, ckm_canonical)` automatically when IMP-2 reports a mirror, and fold its output (added/removed at-codes, cardinality/occurrences/binding changes, plus IMP-3 checksum status) into the review packet.
- **Acceptance:** A review of a mirrored archetype automatically includes a drift table and checksum status without the reviewer issuing a separate command.

---

### IMP-5 — Severity-annotated, machine-readable rules manifest (P2)

- **Target:** `openehr-assistant-mcp` (the `archetypes/rules` guide data) + `openehr-assistant-plugin` (`archetype-lint` skill)
- **Type:** guide/data edit + skill edit
- **Problem:** `archetype-lint` advertises ERROR/WARNING/INFO per rule, but the `rules` guide does not encode the severity mapping. Reviewers assign severity by judgement, so two runs can disagree.
- **Evidence:** I classified the untranslated Swedish label (E7) as WARNING and the English typos as INFO entirely on my own judgement; nothing in the rulebook fixed those levels.
- **Proposed change:** Add a severity column to each rule (A1, C2, D2, E7, …) and expose a machine-readable manifest (JSON) the linter consumes. Keep the prose guide human-readable but generated from / consistent with the manifest.
- **Acceptance:** `archetype-lint` output cites a severity that is traceable to the manifest, not invented; the same file yields identical severities across runs.

---

### IMP-6 — Add a gold-standard COMPOSITION/container example (P2)

- **Target:** `openehr-assistant-mcp` (`resources/examples/archetypes/`)
- **Type:** data (curated example)
- **Problem:** `examples_search(kind="archetypes")` has no reference COMPOSITION. The "generic document container with an Extension slot, unconstrained content, fixed `category`" pattern has no prior-art exemplar to compare against, so idiom judgements rely on the model's training rather than a curated reference.
- **Evidence:** During the review I confirmed the open `/.*/` Extension slot and unconstrained `content` were idiomatic from the anti-patterns guide and memory — there was no COMPOSITION example to anchor on.
- **Proposed change:** Add a curated COMPOSITION example (e.g. a report/encounter/health-summary container) to the examples corpus at `openehr://examples/archetypes/{name}`, with a metadata header explaining the container idiom, the Extension-slot pattern, and the `category` fixing decision.
- **Acceptance:** `examples_search("composition container")` returns the new entry; its notes explain when unconstrained `content` is correct.

---

### IMP-7 — Surface archetype-impact in the review packet (P3)

- **Target:** `openehr-assistant-plugin` (`review-remediate.md`)
- **Type:** skill edit
- **Problem:** Judging whether a change (e.g. localising a label, constraining `content`) is *safe* requires knowing which workspace templates/AQL reference the archetype. `/archetype-impact` does this but is a separate manual step, so impact is absent from review output.
- **Evidence:** My recommendation on the Swedish-label fix could state the upstream/local trade-off but not the concrete blast radius within this workspace.
- **Proposed change:** Have the review pipeline run an impact scan and include a short "referenced by N templates / M AQL files" line in the packet, with a path-stability warning when a proposed fix touches paths/at-codes.
- **Acceptance:** Reviews of a widely-referenced archetype show the reference count and flag any path-affecting fix.

---

### IMP-8 — Reconcile the "mandatory guides" lists (P3)

- **Target:** `openehr-assistant-plugin` (`archetype-authoring` skill)
- **Type:** skill edit
- **Problem:** Step 1 declares loading `principles` + `rules` + `adl-syntax` MANDATORY, while the `review-remediate.md` reference says load `rules` + `checklist` + `anti-patterns`. The two lists diverge, leaving the "mandatory" set ambiguous for a review task.
- **Evidence:** I followed the review-specific list (correct for review) and did not load `principles`/`adl-syntax`; both skill texts claim authority.
- **Proposed change:** Make the guide-loading set *task-conditional* and consistent: state explicitly that a **review** loads `rules` + `checklist` + `anti-patterns` (syntax/principles optional unless syntax errors are suspected), while **authoring** loads `principles` + `rules` + `adl-syntax`. Cross-link the two so they don't contradict.
- **Acceptance:** The skill no longer presents two conflicting "mandatory" lists; the review path and authoring path each name one unambiguous guide set.

---

## Priority summary

| ID | Title | Target | Priority |
|----|-------|--------|----------|
| IMP-1 | Real ADL/AOM validator tool | mcp | P1 |
| IMP-2 | Provenance / CKM-mirror detection | mcp | P1 |
| IMP-3 | Checksum recompute / verify | mcp | P1 |
| IMP-4 | Wire semantic-diff into review pipeline | plugin (+mcp) | P2 |
| IMP-5 | Severity-annotated rules manifest | mcp + plugin | P2 |
| IMP-6 | Gold-standard COMPOSITION example | mcp | P2 |
| IMP-7 | Impact scan in review packet | plugin | P3 |
| IMP-8 | Reconcile mandatory-guides lists | plugin | P3 |

The three P1 items (validator, provenance, checksum) together convert the most error-prone, manually-reasoned parts of an archetype review into deterministic tool calls — and would have *verified* rather than *inferred* the central conclusion of the originating session.
