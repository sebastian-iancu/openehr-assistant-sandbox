# Design — "Rapportage" OET template (GGZ, NL)

- **Date:** 2026-06-09
- **Branch:** `try-2-2026-06-09`
- **Status:** Approved (design), pending implementation

## Context

In Dutch GGZ (mental healthcare), *Rapportage* spans a range of reporting artifacts —
from a single free-text note about a contact (*generieke rapportage*), through
high-volume daily/shift narratives (*dagrapportage / zorgrapportage*), to
treatment progress notes tied to goals (*behandel-/voortgangsrapportage*, often
SOEP-structured).

This work targets the **generieke rapportage**: one free-text report per contact,
context-agnostic, broadest reuse. It is, as the requester intuited, a *clinical
synopsis recorded against an encounter*.

## Decisions (locked during brainstorming)

1. **Scope:** generic rapportage — a narrative report per contact. Not daily/shift,
   not treatment-goal-linked. (YAGNI on the richer flavours.)
2. **Reuse-first:** reuse both published archetypes **as-is**, unmodified on disk:
   - `openEHR-EHR-COMPOSITION.encounter.v1` — contact/event wrapper.
   - `openEHR-EHR-EVALUATION.clinical_synopsis.v1` — narrative carrier.
   Consequence: **no new archetypes** are authored. The deliverable is a template only.
   This is the openEHR-correct, reuse-first outcome.
3. **Artifact format:** an **OET template source** (`local/Rapportage.oet`), the
   hand-authorable openEHR Template XML. Chosen over the repo's tool-generated
   canonical `.t.json` to avoid hand-fabricating content-addressed `MD5-CAM` hashes
   (which CLAUDE.md forbids). OET carries its own `id`/`uid`; no MD5 concern.

### Repo finding that informed the format decision

All existing `local/*.t.json` are ADL-Designer-generated **canonical TEMPLATE JSON**
(AOM-style `rmTypeName`/`attributes`/`nodeId`, a `t_<concept>` root specialising one
parent archetype, `MD5-CAM` hashes, `lifecycleState = unmanaged`). `Vital signs.t.json`
is rooted at `COMPOSITION.encounter` but is an **empty shell** (COMPOSITION +
EVENT_CONTEXT only, no content entries). So the repo has **no worked example of a
COMPOSITION template that fills `content` with an entry archetype** — Rapportage
extends that pattern. Hand-producing valid canonical `.t.json` (with correct MD5) is
not feasible in this environment; OET is the proper authorable source.

## Design

### Structure

```
Rapportage  (template)
└─ COMPOSITION  openEHR-EHR-COMPOSITION.encounter.v1   ← root (category = event/433)
   └─ content
      └─ EVALUATION  openEHR-EHR-EVALUATION.clinical_synopsis.v1
         └─ ITEM_TREE [at0001]
            └─ ELEMENT [at0002] "Synopsis"  → relabelled "Rapportage", DV_TEXT, mandatory
```

The *who / when / role* (composer, contact datetime, care-professional
participations) stays in the Reference Model / `EVENT_CONTEXT` — **not** modelled in
the template.

### Reused components (unchanged on disk)

| Archetype | Role | Relevant nodes |
|-----------|------|----------------|
| `COMPOSITION.encounter.v1` | event/contact wrapper, `category = event` (openehr::433) | `other_context` extension slot `[at0002]` |
| `EVALUATION.clinical_synopsis.v1` | narrative carrier | `data/ITEM_TREE[at0001]/ELEMENT[at0002]` "Synopsis" (DV_TEXT); `protocol/ITEM_TREE[at0003]` extension CLUSTER `[at0004]` |

### Template-level constraints (approved defaults)

| # | Decision | Setting | Rationale |
|---|----------|---------|-----------|
| a | Synopsis element | rename label → **"Rapportage"**, occurrences **1..1 (mandatory)** | An empty rapportage is meaningless. |
| b | Template language | **`nl` primary**, `en` fallback | GGZ NL audience; template-local Dutch labels. |
| c | clinical_synopsis `protocol` extension slot `[at0004]` | **close (0..0)** | Lean generic form. |
| d | encounter `other_context` extension slot `[at0002]` | **close (0..0)** | Lean generic form. |
| e | Additional content entries (e.g. reason_for_encounter) | **none** | "Generic" = just the narrative. |
| f | Identifiers | template_id **`Rapportage`**, file **`local/Rapportage.oet`**, fresh `uid` | Matches concept; OET has its own id/uid. |

## Validation approach

1. Structural cross-check of the OET against the two source archetypes' node IDs
   (`COMPOSITION.encounter` `at0000`/`at0002`; `clinical_synopsis` `at0000`/`at0001`/`at0002`/`at0003`/`at0004`).
2. Run the openehr-assistant MCP explain/lint tooling to confirm slot fills and path
   references resolve.
3. Confirm the Dutch term overlay (template-local names) is well-formed.
4. No runtime/MD5 generation required for OET source.

## Out of scope (YAGNI — deferred)

- Report-type (`soort rapportage`) and `onderwerp`/topic fields.
- SOEP (Subjectief/Objectief/Evaluatie/Plan) structure.
- Links to `behandeldoel` / `probleem`.
- Multiple content entries.

These belong to a future *behandelrapportage* template/archetype if ever needed.
