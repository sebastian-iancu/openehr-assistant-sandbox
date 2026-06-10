# Six-Minute Walk Test (6MWT) — archetype + template design

**Date:** 2026-06-09
**Status:** Approved design — ready for implementation planning
**Author:** Sebastian Iancu (with Claude Code)

## 1. Purpose & scope

Model a **Six-Minute Walk Test (6MWT)** as a reusable openEHR artifact set: one new
`OBSERVATION` archetype plus a `COMPOSITION`-rooted template that aggregates it with the
standard vital-sign observations recorded around the test.

The 6MWT is a standardized sub-maximal exercise test (ATS 2002; ATS/ERS 2014) in which a
patient walks as far as possible in six minutes on a flat course. Its primary outcome is the
**6-minute walk distance (6MWD)** in metres, interpreted alongside heart rate, blood pressure,
peripheral oxygen saturation, and perceived exertion (Borg) measured at rest and after the walk.

**In scope:** the new 6MWT OBSERVATION (single summary event), supplemental-oxygen capture, and a
template composing the 6MWT with reused pulse / blood pressure / pulse oximetry observations.

**Out of scope (deliberately deferred):** predicted 6MWD and % predicted, stops/rests and
early-termination reasons, course-length / assistive-device / encouragement metadata, and any
per-minute or per-lap intra-test event series. These were considered and excluded to keep the
first iteration lean; the extensibility slots below leave room to add them later.

## 2. Design decisions (confirmed with user)

| Decision | Choice | Rationale |
|---|---|---|
| Reuse strategy | Reuse-first, grounded on a CKM survey | No published 6MWT exists in CKM, so a NEW observation is justified; vitals are reused. |
| Architecture | COMPOSITION template → dedicated OBSERVATION + reused vitals | Most RM-idiomatic and reusable; avoids duplicating vital-sign concepts. |
| Timing granularity | Single summary event in the 6MWT OBSERVATION | Captures walk summary (distance, nadir SpO₂, peak HR, end-of-test Borg); pre/post spot readings live in the reused vitals observations. |
| Ancillary data | Supplemental O₂ during the test only | Other ancillaries deferred (see §1). |
| Borg modelling | Embedded `DV_ORDINAL` 0–10 | Canonical `CLUSTER.level_of_exertion` is only DRAFT/alpha; embedding keeps the model self-contained. ADL 1.4 ordinals are integer-only — the modified-Borg 0.5 step is dropped. |
| Composition root | `COMPOSITION.report-result.v1` | Semantically precise for a diagnostic test report (diverges from the existing `encounter`-based Vital signs template, by design). |
| Template output | Valid `.t.json`, hashes flagged for regen | Mirror `Vital signs.t.json`; fresh `uid`, but `MD5-CAM` / `buildUid` left as marked placeholders for Archetype Designer to regenerate (per CLAUDE.md: do not hand-fabricate hashes). |

## 3. Reuse map (verified against CKM, 2026-06-09)

| Building block | CKM archetype | State | Verdict |
|---|---|---|---|
| Heart rate / pulse | `openEHR-EHR-OBSERVATION.pulse.v2` (rev 2.0.9, Vital signs) | PUBLISHED | REUSE (template content) |
| Blood pressure | `openEHR-EHR-OBSERVATION.blood_pressure.v2` (rev 2.0.16, Vital signs) | PUBLISHED | REUSE (template content) |
| Pulse oximetry / SpO₂ | `openEHR-EHR-OBSERVATION.pulse_oximetry.v1` (rev 1.1.6, Vital signs) | PUBLISHED | REUSE (template content) |
| Supplemental O₂ (device + flow + FiO₂) | `openEHR-EHR-CLUSTER.inspired_oxygen.v1` (rev 1.0.2) | PUBLISHED | REUSE (slot fill in OBSERVATION state) |
| Composition root | `openEHR-EHR-COMPOSITION.report-result.v1` (rev 1.0.8) | PUBLISHED | REUSE (template root) |
| Borg / perceived exertion | `openEHR-EHR-CLUSTER.level_of_exertion.v0` | DRAFT (alpha) | NOT used — embed DV_ORDINAL instead |
| 6-minute walk test itself | none (closest precedent: `OBSERVATION.timed_25_foot_walk.v1`, PUBLISHED) | — | NEW |

Reused observations are referenced by CKM id in the template and **not vendored** into `local/`,
consistent with how `Vital signs.t.json` already references `blood_pressure`, `pulse`, etc.

## 4. New archetype — `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0`

- **adl_version:** 1.4; **version:** `v0` (draft), fresh `uid` in the `archetype (...)` header.
- **Filename** must match the inner archetype id on line 2.
- **Structural precedent:** the published `OBSERVATION.timed_25_foot_walk.v1` (timed functional walk test).

### 4.1 Structure (single summary event)

```
OBSERVATION[at0000]  "Six-minute walk test"
└─ data: HISTORY[at0001]
   └─ events: EVENT[at0002] occurrences matches {1..1}   "6MWT summary"
      ├─ data: ITEM_TREE[at0003]
      │  ├─ ELEMENT[at0004] 1..1  "Distance walked"            DV_QUANTITY  units=m, min 0   ← 6MWD, primary
      │  ├─ ELEMENT[at0005] 0..1  "Laps completed"             DV_COUNT     min 0
      │  ├─ ELEMENT[at0006] 0..1  "Lowest SpO₂ during test"    DV_QUANTITY  units=%, 0..100
      │  ├─ ELEMENT[at0007] 0..1  "Peak heart rate during test" DV_QUANTITY units=/min, min 0
      │  ├─ ELEMENT[at0008] 0..1  "Borg dyspnoea (end of test)" DV_ORDINAL  0–10 (modified Borg CR10)
      │  ├─ ELEMENT[at0009] 0..1  "Borg fatigue (end of test)"  DV_ORDINAL  0–10 (modified Borg CR10)
      │  ├─ ELEMENT[at0010] 0..1  "Test completed"             DV_BOOLEAN
      │  ├─ ELEMENT[at0011] 0..1  "Clinical interpretation"    DV_TEXT
      │  └─ ELEMENT[at0012] 0..1  "Comment"                    DV_TEXT
      └─ state: ITEM_TREE[at0013]   (EVENT state)
         └─ ARCHETYPE_SLOT[at0014] 0..1  "Supplemental oxygen"  includes CLUSTER.inspired_oxygen.v1
└─ protocol: ITEM_TREE[at0015]
   └─ ARCHETYPE_SLOT[at0016] 0..*  "Extension"  includes any CLUSTER  (extensibility tail)
```

### 4.2 Borg CR10 ordinal value set (used for at0008 and at0009)

Each ordinal element defines its own local symbol codes (ADL 1.4 requires per-element value codes).
Anchor labels (modified Borg CR10):

| Value | Symbol label |
|---|---|
| 0 | Nothing at all |
| 1 | Very slight |
| 2 | Slight |
| 3 | Moderate |
| 4 | Somewhat severe |
| 5 | Severe |
| 6 | — |
| 7 | Very severe |
| 8 | — |
| 9 | Very, very severe (almost maximal) |
| 10 | Maximal |

Known limitation: the original modified-Borg 0.5 step cannot be represented in an ADL 1.4
`DV_ORDINAL`; it is intentionally dropped. (If 0.5 is ever required, revisit with `DV_SCALE` under
a newer RM release.)

### 4.3 Notes
- `Lowest SpO₂` and `Peak heart rate` here are **walk summary** values (nadir / peak during
  ambulation), distinct from the discrete resting/recovery readings carried by the reused
  `pulse_oximetry` / `pulse` observations in the template. The mild overlap is intentional and
  clinically meaningful.
- Supplemental O₂ is placed in **EVENT state**, consistent with how vital-sign archetypes (e.g.
  pulse oximetry) model inspired-oxygen context.
- All at-codes need full `term_definitions` (text + description) in the ontology; `en` original
  language. Add other languages only if the user requests translation.

## 5. New template — `Six minute walk test.t.json`

Format: openEHR Web Template / Archetype-Designer **CAM differential** export, mirroring
`local/Vital signs.t.json`.

- `@type`: `TEMPLATE`; `templateId`: `"Six minute walk test"`.
- `parentArchetypeId`: `openEHR-EHR-COMPOSITION.report-result.v1`.
- `archetypeId.value`: `openEHR-EHR-COMPOSITION.t_six_minute_walk_test.v1` (template-local `t_` id,
  matching the `t_encounter` pattern in the Vital signs template).
- `adlVersion`: `"1.4"`; `originalLanguage`: `en`.
- `uid`: freshly generated.
- `otherDetails.MD5-CAM-1.0.1`, `otherDetails.PARENT:MD5-CAM-1.0.1`, and `buildUid`:
  **placeholders clearly marked `REGENERATE_IN_ARCHETYPE_DESIGNER`** — not hand-fabricated.

### 5.1 Composition content (slots + constraints)

| Content item | Archetype | Occurrences | Narrowing |
|---|---|---|---|
| 6-minute walk test | `OBSERVATION.six_minute_walk_test.v0` | 1..1 | Distance mandatory; metric units only. |
| Heart rate | `OBSERVATION.pulse.v2` | 0..1 | Allow ≥2 events (resting + post-test); rate in /min. |
| Blood pressure | `OBSERVATION.blood_pressure.v2` | 0..1 | mmHg; allow resting + post-test events. |
| Pulse oximetry | `OBSERVATION.pulse_oximetry.v1` | 0..1 | SpO₂ %; allow resting + post-test events. |

Non-metric unit alternatives constrained out where the source archetype offers them (same approach
as the Vital signs template's purpose statement).

## 6. Implementation order (for the plan)

1. Author `OBSERVATION.six_minute_walk_test.v0.adl` (structure → ontology/terminology → Borg value sets).
2. Lint it (`archetype-lint`, STRICT) and remediate to zero ERRORs.
3. Author `Six minute walk test.t.json` (root, content slots, narrowing, placeholders for hashes).
4. Cross-check: every template archetype id resolves (reused CKM ids + the new local OBSERVATION).
5. (User) regenerate template in Archetype Designer to produce authentic `uid` / MD5-CAM / `buildUid`.

## 7. Acceptance criteria

- `OBSERVATION.six_minute_walk_test.v0.adl` parses as ADL 1.4, passes STRICT lint with zero ERRORs,
  every at-code defined, single mandatory `Distance walked`, supplemental-O₂ slot bound to
  `CLUSTER.inspired_oxygen.v1`.
- `Six minute walk test.t.json` is structurally valid against the `Vital signs.t.json` shape, roots
  `report-result.v1`, composes the four observations with the occurrences above, and marks all
  tool-generated hashes/uids as regeneration placeholders.
- No reused CKM archetype is copied into `local/`.

## 8. Open items / risks

- **MD5-CAM hashing** is tool-owned; the delivered template is structurally valid but must be
  re-exported from Archetype Designer before production use.
- **Borg 0.5 step** not representable in ADL 1.4 (accepted limitation, see §4.2).
- **Resting vs post-test events** in the reused vitals are modelled via each archetype's native
  event structure; if a form needs two explicitly-named clones ("Resting" / "Post-test"), that is a
  template-elaboration follow-up, not a blocker.
