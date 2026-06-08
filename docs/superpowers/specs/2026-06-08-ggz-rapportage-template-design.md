# Design — "GGZ Rapportage" template (v1)

- **Date:** 2026-06-08
- **Status:** Approved design (pre-implementation)
- **Author:** Sebastian Iancu (with Claude Code)
- **Artifact type:** openEHR template (composes existing PUBLISHED archetypes)

## 1. Background

In the Dutch healthcare landscape, *Rapportage* is the running record a care
provider keeps about a client. In **GGZ (mental healthcare)** this is the
**decursus**: a chronological series of contact/session notes documenting the
course of treatment.

The user was unsure how *Rapportage* maps onto openEHR and hypothesised "a
clinical synopsis combined with an encounter". That hypothesis is correct and is
already supported by archetypes present in this workspace
(`COMPOSITION.encounter.v1`, `EVALUATION.clinical_synopsis.v1`).

## 2. Scope

### In scope (v1)

A template for **one GGZ treatment contact** = one *event* COMPOSITION. The
decursus itself is **not** a single persistent document; it is the chronological
set of these compositions for a patient, reconstructed by an AQL query.

### Out of scope (v2 roadmap)

- Coded **contact type / modality** (face-to-face / telephone / video / written /
  group) — deferred until a national valueset is selected (e.g. GGZ / ZPM
  registration). Requires a small holder archetype (see Decision A).
- Discrete, individually-queryable **SOEP** fields (Subjectief / Objectief /
  Evaluatie / Plan) — would be added via a `clinical_synopsis` specialization.
- **Behandeldoel (treatment-goal) linkage** — via RM `LINK` or
  `EVALUATION.goal.v1`.

## 3. Reuse-first survey (international CKM, live, 2026-06-08)

| Building block | Best CKM fit | Status | Verdict |
|---|---|---|---|
| Event container | `openEHR-EHR-COMPOSITION.encounter.v1` (rev 1.0.12) | PUBLISHED | **Reuse** — no dedicated "progress note" composition exists |
| Narrative body | `openEHR-EHR-EVALUATION.clinical_synopsis.v1` (rev 1.0.4) | PUBLISHED | **Reuse** — top hit; the standard narrative summary |
| Reason for contact | `openEHR-EHR-EVALUATION.reason_for_encounter.v1` (rev 1.0.3) | PUBLISHED | **Reuse** (optional) — reden van contact |
| SOEP / SOAP structure | *none* | — | No off-the-shelf international archetype |
| Treatment goal | `openEHR-EHR-EVALUATION.goal.v1` | PUBLISHED | Deferred to v2 |
| Contact type / modality | *none* | — | Deferred to v2 (Decision A) |

Conclusion: the international CKM provides generic building blocks only. A
SOEP-structured / behandeldoel model belongs to the **NL national CKM / Nictiz
zib** space and is intentionally out of v1 scope. All three reused archetypes are
PUBLISHED and already present locally.

## 4. Design decisions

| # | Decision | Choice | Rationale |
|---|---|---|---|
| Granularity | One composition per contact, or one document? | **One *event* COMPOSITION per contact** | Idiomatic openEHR; decursus = AQL view across contacts |
| Body structure | Free narrative vs SOEP vs per-goal vs hybrid | **Narrative + optional structure** | User choice; keeps it usable, allows richer entries later |
| Approach | A (lean) / B (specialized synopsis) / C (new archetype) | **A — lean reuse, 0 new archetypes** | Maximum reuse; SOEP as writing convention for v1 |
| Goal linkage | RM LINK / structured field / both / skip | **Skip for v1** | YAGNI; no concrete need yet |
| Decision A — contact type | tiny cluster / setting-only / defer | **Defer to v2** | Avoids inventing a holder before a national valueset exists |
| Decision B — deliverable | .t.json / .oet / both | **`.oet` only** (revised 2026-06-09) | OET is the correct hand-authorable source; user opted to drop the tool-generated `.t.json` (which carries non-hand-authorable `MD5-CAM` hashes) |

## 5. Template structure

```
COMPOSITION.encounter          [category = event (openehr::433)]        "GGZ Rapportage"
├─ context  (EVENT_CONTEXT — RM only, no archetype):
│    • start_time   → datum/tijd van het contact                         (required)
│    • setting      → "mental healthcare" (openehr::802)                  (default)
│    • composer + participations → behandelaar + rol (reporter)          (RM only)
└─ content:
     ├─ EVALUATION.clinical_synopsis    → de Rapportage (Synopsis 1..1)        [PUBLISHED reuse]
     │      SOEP = schrijfconventie (S/O/E/P als kopjes in de vrije tekst)
     └─ EVALUATION.reason_for_encounter → reden van contact (0..1, optional)   [PUBLISHED reuse]
```

### Per-node detail

- **Root `COMPOSITION.encounter`** — `category` fixed to coded `event`
  (`openehr::433`). Title/name presented to the clinician: "GGZ Rapportage".
- **`context.start_time`** — date/time of the contact; required.
- **`context.setting`** — defaulted to `mental healthcare` (`openehr::802`,
  openEHR `setting` group). May be overridden by the host system.
- **`context.participations` + `composer`** — the reporting clinician
  (behandelaar) and role. Modelled purely in the RM; no archetype. Expected
  participation `function` values to document for implementers: *behandelaar*,
  *regiebehandelaar* (exact codes/valueset to be agreed by the implementing
  system).
- **`content[0]` = `EVALUATION.clinical_synopsis`** — the report body. The
  `Synopsis` element (free text, `DV_TEXT`) carries the narrative, constrained to
  `1..1` (exactly one report per contact). SOEP is a documented **writing
  convention** in v1 (author writes under S/O/E/P headings); it is not modelled
  as discrete data.
- **`content[1]` = `EVALUATION.reason_for_encounter`** — optional (`0..1`),
  captures *reden van contact*.

## 6. Terminology bindings (grounded, not fabricated)

| Concept | openEHR Terminology | Group |
|---|---|---|
| Composition category | `433` = *event* | `composition_category` |
| Care setting | `802` = *mental healthcare* | `setting` |

Both resolved live via `terminology_resolve` on 2026-06-08.

## 7. Deliverables

**`local/GGZ Rapportage.oet`** — hand-authorable OET (Ocean Template) XML, the
single deliverable. It:

- roots on `openEHR-EHR-COMPOSITION.encounter.v1` (`concept_name="GGZ Rapportage"`);
- slots `EVALUATION.clinical_synopsis.v1` into `/content` as the report
  (`name="Rapportage"`), with its `Synopsis` element constrained to `1..1`;
- slots `EVALUATION.reason_for_encounter.v1` into `/content` as optional
  (`min=0 max=1`, `name="Reden van contact"`);
- documents the intended care `setting` = *mental healthcare* (`openehr::802`)
  in `other_details` (the `setting` value itself is set by the recording system;
  fixing it to a single code via a context `Rule` is a v1.1 refinement to
  validate against Ocean/Archetype Designer first);
- carries a freshly-minted UUID in `<id>` and **omits** `<integrity_checks>`
  digests (these `MD5-CAM` values are tool-generated by Ocean/Archetype Designer
  and are never hand-fabricated; the toolchain adds them on import).

> The `.t.json` differential-template JSON (the other files in `local/`) is
> dropped: it is normally **generated** by ADL/Archetype Designer and carries
> tool-computed `MD5-CAM`/`buildUid` values that cannot be hand-authored
> correctly. The `.oet` is the correct design-time source; an OPT/`.t.json` can
> be generated from it downstream if ever needed.

## 8. Conventions

- **Filenames:** spaces permitted (matches existing `Vital signs.t.json` etc.).
- **Language:** default `nl`, with `en` provided. `encounter.v1` already ships an
  `nl` translation; `clinical_synopsis.v1` and `reason_for_encounter.v1` do **not**
  — `nl` term text for any node surfaced in the template must be supplied at the
  template level (or the source archetypes translated separately, out of scope).
- **Version:** first version `v1`.

## 9. Success criteria

1. The template validates against the two PUBLISHED archetypes (no constraint
   violations).
2. A FLAT example composition for a single GGZ contact populates the narrative,
   an optional reason for contact, the contact date/time, and the reporter
   (composer/participation).
3. An AQL query retrieves a patient's decursus — all `encounter` compositions
   containing a `clinical_synopsis`, ordered chronologically by
   `context/start_time`.

## 10. References

- `openEHR-EHR-COMPOSITION.encounter.v1` — CKM CID 1013.1.120 (PUBLISHED)
- `openEHR-EHR-EVALUATION.clinical_synopsis.v1` — CKM CID 1013.1.409 (PUBLISHED)
- `openEHR-EHR-EVALUATION.reason_for_encounter.v1` — CKM CID 1013.1.290 (PUBLISHED)
- Local copies of all three archetypes exist in `local/`.
