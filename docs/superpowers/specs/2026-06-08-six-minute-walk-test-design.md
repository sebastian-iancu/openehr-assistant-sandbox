# 6-Minute Walk Test (6MWT) — minimal archetype + template design

**Date:** 2026-06-08
**Status:** Approved (brainstorming) — pending implementation plan
**Repository:** openehr-assistant-sandbox (content-only: `*.adl` archetypes + `*.t.json` templates in `local/`)

## 1. Purpose

Model a 6-Minute Walk Test (6MWT) as a reusable openEHR artifact set. The 6MWT is a
sub-maximal functional exercise-capacity test in which a patient walks as far as possible
in 6 minutes on a flat course; total distance walked is the primary outcome.

This design targets a **distance-only minimal** artifact set — a teaching/demo-grade,
self-contained result container — not the full ATS/ERS instrumented protocol. The model
is forward-compatible: richer payload (paired vitals, supplemental O₂, stop episodes) can
be added later without breaking `v0` data.

## 2. Reuse-first analysis (CKM survey)

A CKM reuse survey confirmed there is **no 6MWT archetype and no 6MWT template** published
in CKM (searched: "six minute walk", "walk test", "exercise test", "functional exercise
capacity", "exercise tolerance", "walking distance"). The core test container must be
authored new.

Mature, published archetypes exist for the *surrounding* payload and are deliberately left
**out of scope** for this minimal version (each is a one-line slot/sibling addition later):

| Concept | CKM archetype | Status | Decision (this version) |
|---|---|---|---|
| SpO₂ pre/post | `OBSERVATION.pulse_oximetry.v1` | PUBLISHED | Out of scope (future) |
| Heart rate pre/post | `OBSERVATION.pulse.v2` | PUBLISHED | Out of scope (future) |
| Blood pressure pre/post | `OBSERVATION.blood_pressure.v2` | PUBLISHED | Out of scope (future) |
| Supplemental O₂ | `CLUSTER.inspired_oxygen.v1` | PUBLISHED | Out of scope (future) |
| Walking aids | `CLUSTER.device.v1` | PUBLISHED | Replaced by inline coded text (minimal) |
| Exertion/Borg context | `CLUSTER.level_of_exertion.v0` | in_development | Replaced by inline ordinal (minimal) |
| Overall interpretation | `EVALUATION.clinical_synopsis.v1` | PUBLISHED | Out of scope (future) |

Structural style references (patterns mirrored, not specialized): `OBSERVATION.timed_25_foot_walk.v1`
(timed-walk idioms), plus this repo's own `OBSERVATION.ecg_result.v1` and
`OBSERVATION.laboratory_test_result.v1` (result-container shape).

## 3. Decisions

- **Scope:** distance-only minimal — distance walked + duration + walking aid, one OBSERVATION.
- **Borg:** included as a *single optional inline* `DV_ORDINAL` (modified Borg CR10, 0–10),
  not a separate CLUSTER and not specialized from `level_of_exertion.v0`.
- **Walking aid:** inline `DV_CODED_TEXT` over a local value set (not reused `CLUSTER.device.v1`).
- **Data shape:** single point-event summary (Approach A), not multi-event interval (Approach B).
  A→B is a non-breaking evolution (relax event occurrences) so B is not paid for now.
- **No external slots, no nested CLUSTERs** — fully self-contained per the minimal/inline preference.

## 4. Artifacts

Exactly two files in `local/` (one archetype + one template):

### 4.1 `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

ADL 1.4 (`adl_version=1.4`), fresh `uid` in the `archetype (...)` header, inner archetype
identifier `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0` matching the filename, `v0` (draft)
lifecycle. Fixed ADL section order (archetype header → concept → language → description →
definition → ontology/terminology), tab-indented.

Definition structure:

```
OBSERVATION[at0000]  -- 6 Minute Walk Test
  data: HISTORY[at0001]
    events: POINT_EVENT[at0002] occurrences matches {1..1}  -- End of test
      data: ITEM_TREE[at0003]
        ├ [at0004] Distance walked      DV_QUANTITY  units=m, magnitude >= 0   (1..1, mandatory)
        ├ [at0005] Duration walked      DV_DURATION  default PT6M               (0..1)
        ├ [at0006] Number of laps       DV_COUNT     magnitude >= 0            (0..1)
        └ [at0007] Borg breathlessness  DV_ORDINAL   modified Borg CR10 0..10  (0..1, optional)
  state: ITEM_TREE[at0008]
        └ [at0009] Walking aid used     DV_CODED_TEXT  local value set         (0..1)
  protocol: ITEM_TREE[at0010]
        ├ [at0011] Course length        DV_QUANTITY  units=m                   (0..1)
        └ [at0012] Comment              DV_TEXT                                 (0..1)
```

at-code allocation (terminology / `term_definitions`, language `en`):

| Code | Rubric | Type |
|---|---|---|
| at0000 | 6 Minute Walk Test | OBSERVATION root |
| at0001 | History | HISTORY |
| at0002 | End of test | POINT_EVENT |
| at0003 | Tree | ITEM_TREE (data) |
| at0004 | Distance walked | DV_QUANTITY |
| at0005 | Duration walked | DV_DURATION |
| at0006 | Number of laps | DV_COUNT |
| at0007 | Borg breathlessness score | DV_ORDINAL |
| at0008 | Tree | ITEM_TREE (state) |
| at0009 | Walking aid used | DV_CODED_TEXT |
| at0010 | Tree | ITEM_TREE (protocol) |
| at0011 | Course length | DV_QUANTITY |
| at0012 | Comment | DV_TEXT |

Local value sets (allocated `at` codes, defined in `term_definitions` + listed in `value_sets`):

- **Walking aid used (at0009):** None (assumed value) · Walking stick/cane · Crutches ·
  Walking frame/rollator · Wheelchair · Other.
- **Borg breathlessness (at0007):** modified Borg CR10 ordinal — values 0–10 with standard
  rubrics (0 Nothing at all, 0.5 Very very slight, 1 Very slight, 2 Slight, 3 Moderate,
  4 Somewhat severe, 5 Severe, 7 Very severe, 9 Very very severe, 10 Maximal). Implementation
  note: `DV_ORDINAL` requires integer values, so the 0.5 step of the canonical CR10 scale is
  dropped; integer 0–10 ordinal is used. (Revisit if half-steps are clinically required.)

### 4.2 `local/6 Minute Walk Test.t.json`

openEHR Web Template (simplified) JSON, top-level `"@type": "TEMPLATE"`, fresh `uid`,
following the structure of the existing `local/Vital signs.t.json`.

- `COMPOSITION` rooted on the existing `local/openEHR-EHR-COMPOSITION.encounter.v1.adl`.
- Contains the single new `OBSERVATION.six_minute_walk_test.v0`.
- Minimal constraints only (no narrowing beyond what the archetype already defines).
- MD5 hashes / `uid` are generated by tooling, not hand-fabricated.

## 5. Out of scope

Paired pre/post vitals (pulse_oximetry, pulse, blood_pressure), supplemental O₂
(inspired_oxygen), stop/rest-episode repeatable cluster, early-termination reason set,
symptom list, clinical synopsis, separate Borg/exertion CLUSTER, multi-event interval
splits. All are additive to `v0` and explicitly deferred.

## 6. Validation criteria

- Archetype passes `/archetype-lint` (the 22 normative rules) in this workspace.
- Inner archetype id on line 2 matches the filename.
- Every node has a `term_definitions` entry; `value_sets` cover both coded nodes.
- Template references only archetypes present in `local/` (the new OBSERVATION + encounter COMPOSITION).
