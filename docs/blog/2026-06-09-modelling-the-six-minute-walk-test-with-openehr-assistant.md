# Modelling a 6-Minute Walk Test with the openEHR Assistant — a session retrospective

*2026-06-09 · a working log of how an AI pair-modeller took "help me model a 6MWT" from a one-line request to two committed, lint-clean openEHR artifacts — and where the toolchain still falls short.*

---

## The request

> *"Help me out to model a 6-minute walk test set of templates and archetypes — what do I need?"*

Deceptively small. The 6-Minute Walk Test (6MWT) is a sub-maximal test of functional exercise capacity: the patient walks as far as possible on a flat course in six minutes, and the distance walked is the primary outcome. Clinically it carries a long tail of adjuncts — paired pre/post SpO₂, heart rate, blood pressure, Borg dyspnoea/fatigue ratings, supplemental oxygen, rest/stop episodes, reasons for early termination. A faithful model could sprawl across a dozen archetypes. The real question behind "what do I need?" is therefore a *scoping* question, and getting that wrong in either direction (over-building a research-grade instrument, or under-building an unusable stub) is the main risk.

## How it was solved

The work followed a deliberate pipeline rather than jumping straight to ADL:

1. **Brainstorming first.** Before writing anything, intent was pinned down. The single highest-leverage fork — *fidelity* — was put to the user as a structured choice (full ATS/ERS instrument vs. core test vs. distance-only minimal), alongside how to model the Borg score. The user chose **distance-only minimal** with an **inline** Borg ordinal. That one decision collapsed the design space from ~12 archetypes to **two files**.

2. **Reuse-first survey.** The single most-violated principle in openEHR is *reuse before authoring*. A context-isolated CKM scout ran parallel searches across many phrasings ("six minute walk", "walk test", "exercise test", "functional exercise capacity", …) for both the overall test and every component concept. Headline finding: **no 6MWT archetype or template exists in CKM**, but almost the entire surrounding payload does (mature, published) — `pulse_oximetry`, `pulse`, `blood_pressure`, `inspired_oxygen`, `device`, `clinical_synopsis`, plus `level_of_exertion` as a near-miss for Borg. This is the classic *"author one thin OBSERVATION, compose many reused components"* shape.

3. **Design spec → implementation plan.** A short spec captured the decisions and the exact node/at-code layout; a plan turned it into bite-sized, literal-content tasks with `/archetype-lint` as the test gate.

4. **Subagent-driven execution.** A fresh authoring agent wrote each file from the literal spec; the main session ran the authoritative lint and an independent spec-compliance read between tasks.

The result, committed to `main`:

| Artifact | What it holds |
|---|---|
| `OBSERVATION.six_minute_walk_test.v0.adl` | distance walked (m), duration, laps, inline Borg CR10 ordinal (0–10), walking-aid coded text, course length, comment |
| `6 Minute Walk Test.t.json` | binds the OBSERVATION into the existing `COMPOSITION.encounter.v1` |

Lint verdict: **PASS** — 0 errors, 0 warnings, 2 (intentional) INFOs.

## What the openEHR Assistant MCP, plugin, and skills actually contributed

This was not modelling from memory. Concretely, the toolchain supplied:

- **`ckm-scout` agent (reuse survey).** Turned a vague "does this exist?" into a ranked reuse/specialise/author-new recommendation, with CKM lifecycle states and fit scores. This is what justified authoring *only one* new archetype and reusing the rest — the difference between a defensible model and a reinvented wheel.
- **`terminology_resolve`.** Confirmed the openEHR property code for length (units `m`) is **`[openehr::122]`** instead of guessing it. Small, but exactly the kind of fact that silently corrupts a `C_DV_QUANTITY` when fabricated.
- **`guide_get(archetypes/rules)` + `(archetypes/structural-constraints)`.** Provided the normative rule catalogue (A–K, VARID/VATDF/… tooling codes) that the `/archetype-lint` pass applied — e.g. confirming the `state`/`protocol` placement and the at-code completeness rule (VATDF).
- **`guide_adl_idiom_lookup`.** Confirmed the *inline coded value-set* idiom (`[local:: …]` in `defining_code`) over a separate `value_sets` block — which also happened to match this repo's existing `ecg_result.v1` convention.
- **`examples_search` (archetypes corpus).** Surfaced published reference archetypes (`blood_pressure.v2`, `problem_diagnosis.v1`, …) as structural style anchors.
- **`/archetype-lint` skill.** Applied the 22 normative rules and returned a PASS/FAIL with a violations table — the test gate of the whole exercise.
- **`clinical-modeler` agent.** Authored and committed the local `.adl`/`.t.json` files in isolation from the orchestration context.
- **Superpowers process skills** (`brainstorming`, `writing-plans`, `subagent-driven-development`) — the scaffolding that forced scoping *before* code and review *after* it.

The division of labour is the interesting part: the **plugin/skills carried the openEHR domain knowledge**, while the **superpowers skills carried the engineering discipline**. Neither alone would have produced a scoped, reviewed result.

## Shortcomings and gaps — what was missing or awkward

Honesty matters more than a tidy success story. The friction points:

1. **No template compiler / OPT builder.** The biggest gap. openEHR templates in this repo carry tool-computed `MD5-CAM` and `buildUid` checksums and a full operational definition tree — produced by an Archetype Designer / OPT compiler. The MCP exposes **no tool to compile an archetype set into an operational template, nor to compute those hashes.** Per repo policy these must never be hand-fabricated, so the template was delivered in a *lightweight simplified form* deliberately omitting the hashes. That is correct and honest, but it means the headline deliverable — "a template" — is a stub that still needs a round-trip through a GUI tool to become a true OPT. **A `template_build`/`opt_compile` MCP tool is the single highest-value thing I wished for.**

2. **`/archetype-lint` is rule-application, not a real ADL parser.** The lint is an LLM applying a normative checklist — excellent for catching modelling-intent violations, but it is **not** an AOM/ADL parser that would catch a misplaced brace, an invalid `C_DV_QUANTITY` interval, or a broken path the way ADL Workbench / Archie would. There was no way to get an authoritative *machine* parse of the archetype. A wrapped `adl-validate` (Archie-backed) tool would close this gap and let the lint focus on semantics.

3. **No "pull this CKM archetype into `local/`" action.** The reuse survey *recommended* reusing `pulse_oximetry`, `inspired_oxygen`, etc., but there is no tool to actually fetch a published `.adl` from CKM into the workspace so a template can slot it. Reuse remained advisory; realising it (the full ATS/ERS version) still requires manual download. `ckm_archetype_get` returns content, but there is no curated "materialise into workspace and wire the slot" path.

4. **The authoring agent is walled off from the knowledge tools.** `clinical-modeler` (by design) has no MCP/CKM access, while the agents that *do* have CKM/guide access don't write files. Every grounded fact (property code, idiom, reuse list) had to be marshalled in the main session and hand-carried into the implementer's prompt. It works, but it is a context-shuffling tax; a modelling agent that could both look up and write would be tighter.

5. **`DV_ORDINAL` can't express the Borg 0.5 step.** The modified Borg CR10 scale has a canonical `0.5` ("very, very slight") rung; `DV_ORDINAL` requires integer values, so the model uses an integer 0–10 subset and documents the omission. Faithful, but lossy — a real clinical limitation worth surfacing, not hiding.

6. **No terminology-binding assistant.** The walking-aid set and Borg ordinal are local-coded only. The tooling resolves *openEHR* terminology codes well, but there was no helper to *propose* SNOMED CT / LOINC bindings for these concepts (e.g. a "walking frame" SNOMED concept). For a publication-grade archetype that is the obvious next step, and it was entirely manual.

7. **Environment papercuts.** `uuidgen` was absent (worked around via `/proc/sys/kernel/random/uuid`), and the examples corpus had no 6MWT or a direct inline-ordinal example, so the ordinal syntax leaned on canonical knowledge rather than a retrieved exemplar.

## What I'd reach for next time

- A **`template_build` / OPT-compile** MCP tool (and MD5/buildUid generation) so the template stops being a stub.
- An **Archie-backed `adl_validate`** tool for authoritative parse-level validation, complementing the semantic lint.
- A **CKM "materialise into workspace"** action to make reuse real, not just recommended.
- A **terminology-binding suggester** (SNOMED/LOINC) for coded leaves.
- A **modelling agent with both file-write and MCP-lookup** powers, to end the context hand-off.

## Takeaways

- **Scoping is the model.** One well-framed question (fidelity) did more for the result than any ADL trick. The minimal/extensible decision is what kept this to two files that *grow* into the full instrument rather than two files that have to be thrown away.
- **Reuse-first paid off immediately** — it converted a sprawling concept into "one new OBSERVATION + reuse the rest," and the survey doubles as the roadmap for the next version.
- **Grounded facts beat fluent guesses.** Property code `122`, the inline value-set idiom, the RM placement of `state`/`protocol` — each was *verified*, and each is a place a confident guess would have quietly broken the artifact.
- **The honest gap is the last mile.** Authoring a correct archetype is now well-supported; turning a set of archetypes into a *deployable operational template* still escapes the current toolchain and remains the most valuable thing to build next.

---

*Artifacts from this session:* `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`, `local/6 Minute Walk Test.t.json`, with the design spec and plan under `docs/superpowers/`.
