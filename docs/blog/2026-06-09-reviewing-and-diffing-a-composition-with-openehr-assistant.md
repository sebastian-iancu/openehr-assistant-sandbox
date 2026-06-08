# Reviewing and diffing a COMPOSITION container with the openEHR Assistant — a session retrospective

*2026-06-09 · a working log of how an AI pair-modeller took "design review and QA on `COMPOSITION.health_summary.v1`" through an intent/scope pass, a 22-rule lint, a CKM reuse survey, and finally a **cross-archetype semantic diff** against `COMPOSITION.report.v1` — and what the toolchain did well versus where it still asks the model to improvise.*

---

## The request

The session opened with a quality-and-design question on a single file:

> *"Use your skill and perform a design review and QA on `local/openEHR-EHR-COMPOSITION.health_summary.v1.adl`, spot issues and suggest improvements."*

It then grew, as these sessions do, into two follow-ups:

> *"Are there similar archetypes on CKM?"*
> *"Fetch `COMPOSITION.report.v1` and run a `/archetype-diff`."*

So a one-file review became a three-act walk-through — **review → reuse survey → diff** — on a mature, published artefact. The file under review is not a sandbox draft: it is the openEHR Foundation's canonical **Health summary** COMPOSITION (`lifecycle_state = published`, revision **1.0.3**, `MD5-CAM-1.0.1 = B2D87AA…`, original author Heather Leslie / Atomica Informatics, 2015), carried into `local/` verbatim with its five-language `translations` block (en, sv, nb, pt-br, es).

## How it was solved

Work followed the plugin's documented order of operations — guides and RM facts first, interpretation last.

### 1. Intent, scope, and lint (`/archetype-review` skill)

The `archetype-review` pipeline drove the first two acts: **Stage 1 intent/scope analysis**, then **Stage 2 lint** against the 22 normative rules, with the fix-plan/patch stages gated behind any ERROR.

- **Stage 1 — intent & scope.** Concept: a *generic snapshot-summary container* at COMPOSITION level. RM type `COMPOSITION` is correct (a document/container concept, not an ENTRY). The clinical payload (`content`) is **deliberately unconstrained** — the "populate via template" pattern — while `category` is fixed and a single `other_context` Extension slot is offered. A *must-not-change* list pinned the semantic anchors: `at0000`, `at0001`, `at0002`, the `context/other_context` path, the open `content`, and the `uid`/MD5.
- **Stage 2 — lint.** Guides loaded via MCP (`archetypes/rules`, `structural-constraints`, `anti-patterns`); RM attributes verified against the BMM with `type_specification_get(COMPOSITION)`; the one terminology binding resolved with `terminology_resolve("433")` → **`event`** in the `composition_category` group.

**Verdict (PERMISSIVE): PASS** — 0 ERRORs. Findings were design-judgment and editorial:

| Severity | Finding |
|---|---|
| WARNING | `category` **hard-constrained** to `433\|event\|` — a restriction that rule C3 says belongs in templates. It *is* internally consistent with the "point-in-time snapshot" purpose, so it is a defensible design choice rather than a defect. |
| WARNING | `nb` translation broken — the reader-bullet list drops the "clinicians managing specific aspects" item, leaving the orphan fragment *"som ved diabetes eller gravidiet."*, and `gravidiet` is a typo for `graviditet` (rule E6). |
| INFO | English `use` text: typo *"generic containter"* and the awkward *"familiar with the all of the relevant aspects"*. |
| INFO | The open Extension slot (`include archetype_id/value matches {/.*/}`) — flagged for completeness, but this is the **canonical CKM idiom**, intentionally open and type-restricted to CLUSTER. No change. |
| ✓ verified | `other_context` `items cardinality {0..*}` is **correct, not a defect** — its only child is the *optional* Extension slot, so forcing `1..*` would wrongly mandate an extension. Pre-empting a common false positive. |

Because lint PASSed, no fix plan was due. Instead the review surfaced the single substantive **design decision** (`category = event` vs leaving it to the template) and an important **provenance caveat**: this file is a faithful mirror of a published Foundation artefact, so editing the typo or the `nb` text would invalidate `MD5-CAM-1.0.1` and fork the local copy from the custodian — the clinically-correct path is an upstream CKM contribution, not a silent local edit.

### 2. CKM reuse survey

The "similar archetypes?" follow-up was answered with `ckm_archetype_search` across **three phrasings** — `summary`, `document`, `report` — and the results were sorted into two buckets that matter clinically:

- **True structural siblings (COMPOSITION containers):** `report.v1` (the closest twin — a generic document container with the same open-content + Extension-slot pattern), its specialisations `report-result.v1` / `report-procedure.v1`, plus `transfer_summary.v1`, `medication_list.v1`, `encounter.v1`, `request.v1`.
- **"Summary" look-alikes that are *not* structurally similar:** `EVALUATION.clinical_synopsis.v1`, `EVALUATION.substance_use_summary.v1`, `EVALUATION.obstetric_summary.v1`, etc. — these are ENTRY-level *content you place inside* a health summary, not alternatives to the container.

The headline insight: `health_summary` occupies the *generic snapshot-summary* niche alone; `report` is its nearest structural relative.

### 3. Cross-archetype semantic diff (`/archetype-diff` skill)

The third act fetched the twin with `ckm_archetype_get(openEHR-EHR-COMPOSITION.report.v1)` and ran `/archetype-diff`. The skill loaded its `semantic-diff-rubric.md` and the rules guide, then hit its own guard rail: **the rubric refuses two different root concepts** (it is built to classify version bumps of *one* archetype). With intent explicitly confirmed, the comparison was re-framed as a **structural divergence** analysis rather than a patch/minor/major bump.

The result:

- **Shared skeleton.** Both are openEHR-Foundation generic COMPOSITION containers with identical DNA: `category` fixed to `433|event|`, an `EVENT_CONTEXT.other_context` `ITEM_TREE[at0001]`, **unconstrained `content`**, and the canonical open CLUSTER Extension slot.
- **Major-class divergence.** Different root concept; and critically, **`at0002` is repurposed** — a CLUSTER "Extension" slot in `health_summary` versus an `ELEMENT` "Report ID" (`DV_TEXT`) in `report`. Same at-code, same path (`/context/other_context[at0001]/items[at0002]`), different RM type — any AQL written against one breaks against the other.
- **Additive in `report`.** Two optional header ELEMENTs absent from `health_summary` — "Report ID" (`at0002`) and "Status" (`at0005`) — and the Extension slot recoded to `at0006`.
- **Coverage.** `report` carries 16 languages (several still English placeholders) versus `health_summary`'s 5.

**Verdict:** the two are the same architectural pattern with different headers; neither subsumes the other and they share no compatible lineage. If `health_summary` ever needed a status/ID, the clean move is to *specialise `report`* (or add the ELEMENTs in a template), not reconcile the two.

## What the openEHR Assistant MCP, plugin, and skills contributed

### MCP server (`openehr-assistant-mcp`)

| Tool / resource | How it was used |
|---|---|
| `guide_get` | Loaded `archetypes/rules`, `structural-constraints`, `anti-patterns` — the normative backbone that turned "review this" into a checklist-backed, rule-cited audit. |
| `type_specification_get` | BMM-backed verification of `COMPOSITION` attributes (`category`, `context`, `content`) and invariants (`Category_validity`, `Content_valid`) — grounded the "no invented RM attributes" check in the actual model. |
| `terminology_resolve` | Confirmed `433` → `event` in the `composition_category` group — the single binding in the file, resolved cleanly. |
| `ckm_archetype_search` | Three-phrasing reuse survey that surfaced the COMPOSITION container family and the `report` twin with live status/revision. |
| `ckm_archetype_get` | Fetched the full `report.v1` ADL for the diff. |

The split that worked: **guides carried domain methodology, the MCP carried retrieved facts.** `type_specification_get` is the quiet hero — it let the lint assert RM correctness from the BMM instead of memory.

### Plugin skills and commands

| Skill / command | Role in session |
|---|---|
| `/archetype-review` | Staged the work — intent/scope, lint, fix-gating — and produced the *must-not-change* list and the MD5/provenance caveat that kept the review disciplined. |
| `/archetype-diff` | Supplied the `semantic-diff-rubric` (root concept, at-codes, cardinality, occurrences, bindings, slots, translations) — a principled lens that stayed useful even when repurposed for a cross-archetype comparison. |

## Shortcomings — what was missing or awkward

Honest friction log, specific to this session:

### 1. `/archetype-diff` assumes same-archetype versioning

The skill and its rubric are built to classify a **version bump** of one archetype and explicitly *refuse two different root concepts*. But "how does my container differ from the CKM `report` container?" is a legitimate, common question. The skill had to be coerced past its guard rail, and its patch/minor/major + rule-G1 verdict is meaningless across two distinct archetypes. There is no first-class **sibling / cross-archetype compatibility** mode that emits a divergence-and-compatibility report.

### 2. No `ckm_archetype_diff`; the local-vs-CKM check is manual *and* was not closed

Diffing meant `ckm_archetype_get` → feed the LLM diff skill by hand. Worse, the obvious provenance question — *is `local/health_summary.v1` byte-identical to the published CKM revision?* — was never answered, because I diffed against the *twin* (`report`), not against CKM's own `health_summary`. A `ckm_archetype_diff(local_path, identifier)` returning a hash/metadata/definition comparison would make import hygiene a one-step, deterministic check. *(Reconfirms the earlier brief's P1-4 with fresh evidence.)*

### 3. `ckm_archetype_search` recall for an exact concept name

Searching **`health summary`** did **not** return `COMPOSITION.health_summary` in the top 15 — it surfaced `health_risk`, `substance_use_summary`, and other neighbours instead. Finding the structural siblings needed three separate phrasings. An exact concept-name match should rank at or near the top, and an `rm_class` filter (e.g. "COMPOSITION only") would have collapsed three searches into one.

### 4. `ckm_archetype_get` has no projection — full multi-language payload only

`report.v1` came back with **16 language blocks** when the diff needed only the `definition` section plus the English `term_definitions`. A `section=definition` and/or `language=en` projection (or a structure digest) would cut the token cost of every diff and explain workflow dramatically. *(Echoes the earlier brief's `archetype_digest` ask, now also motivated by diffing.)*

### 5. No structural digest → both definitions parsed by hand

To diff, I mentally parsed each `definition` into a node tree (at-codes, RM types, occurrences, slot includes). A machine **structure digest** would make diffs reproducible and remove the parsing step entirely.

### 6. No provenance / MD5 guardrail

Nothing in the toolchain flagged that the local file is a *published-CKM mirror*, nor warned that editing the typo or `nb` text would invalidate `MD5-CAM-1.0.1` and diverge from the custodian. That "don't silently fork canonical" judgment rested entirely on the model. A provenance check (mirror? which revision? would an edit fork it?) belongs in the tooling.

### 7. The review pipeline equates PASS with "nothing to improve"

`/archetype-review` routes a lint PASS straight to the optional review packet, skipping the fix stages. But the user asked to *spot issues and suggest improvements*, and a PASS-with-advisories (real WARNING/INFO items) still has improvements worth structuring. The pipeline lacks a lightweight "advisory remediation" path that proposes fixes for non-ERROR findings without invoking the full fix-plan/patch machinery.

### 8. Deferred MCP tools needed schema-loading before first use

Every MCP tool (`guide_get`, `ckm_archetype_search`, `ckm_archetype_get`, `type_specification_get`, `terminology_resolve`) was a *deferred* tool whose schema had to be fetched via a discovery step before the first call — extra round-trips at the start of each new capability. This is partly a harness concern, but the plugin could ship a "load these tools first" preamble for its review/diff skills.

## What I'd reach for next time

- **`/archetype-diff` cross-archetype mode** — a sibling-comparison verdict (shared skeleton, repurposed codes, additive fields, path collisions) instead of a version bump, with the refusal downgraded to a confirmation when root concepts differ.
- **`ckm_archetype_diff(local_path, identifier)`** — deterministic local-vs-CKM hash/metadata/definition comparison; would have *closed* the `health_summary` provenance question in one call.
- **`ckm_archetype_get` projection** — `section` and `language` parameters (or an `archetype_digest`) to avoid pulling 16 language blocks for a definition-level diff.
- **`ckm_archetype_search` ranking + `rm_class` filter** — exact concept-name hits ranked first; RM-class scoping to find true structural siblings in one search.
- **Provenance / MD5 guard** — detect that a local file mirrors a published CKM archetype and warn before any edit that would invalidate its hash.
- **Review pipeline "advisory remediation"** — structured fix suggestions for WARNING/INFO findings on a PASS, without the ERROR-only fix-plan/patch flow.

## Takeaways

- **Container archetypes review differently from leaf archetypes.** The substantive findings here were *design* (`category = event` vs template-level) and *provenance* (MD5 mirror), not data-type or cardinality defects — the review skill's intent/scope + must-not-change framing is what surfaced them.
- **"Similar" is a two-level question.** The most useful output of the reuse survey was separating COMPOSITION *containers* (real siblings) from same-word EVALUATION *content* (what goes inside) — a distinction the `archetypeId` RM-class encoding made immediate.
- **A diff rubric built for versions still illuminates siblings.** Repurposed past its guard rail, the at-code/cardinality/binding lens cleanly exposed the `at0002` repurposing and the path collision between two distinct containers.
- **The gaps are diff-and-retrieval ergonomics, not modelling knowledge** — cross-archetype diffing, local-vs-CKM verification, payload projection, search recall, and provenance awareness. The clinical content in guides and the BMM-backed RM facts were, again, excellent.

---

*Primary artefact reviewed:* `local/openEHR-EHR-COMPOSITION.health_summary.v1.adl`
*Compared against:* `openEHR-EHR-COMPOSITION.report.v1` (CKM, revision 1.2.4)
*Companion refactor brief:* `docs/improvements/2026-06-09-openehr-assistant-review-and-diff-improvements.md`
