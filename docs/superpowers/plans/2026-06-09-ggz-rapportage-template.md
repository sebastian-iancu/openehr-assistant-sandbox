# GGZ Rapportage Template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Author one hand-written OET (Ocean Template) file, `local/GGZ Rapportage.oet`, modelling a Dutch GGZ *Rapportage* (decursus) as one event `COMPOSITION.encounter` containing a required `clinical_synopsis` narrative and an optional `reason_for_encounter`.

**Architecture:** Pure-reuse openEHR template (Approach A, zero new archetypes). Root = `openEHR-EHR-COMPOSITION.encounter.v1`; `/content` slots the two PUBLISHED EVALUATION archetypes. SOEP is a writing convention (v1); coded contact type and discrete SOEP are deferred to v2. The decursus is reconstructed by AQL across these compositions, not stored as one document.

**Tech Stack:** OET XML (`xmlns="openEHR/v1/Template"`); validation via the `openehr-assistant` skills (`/template-explain`) and `python3` for XML well-formedness. No build/test runner exists in this content-only repo.

**Source spec:** `docs/superpowers/specs/2026-06-08-ggz-rapportage-template-design.md`

---

## Reference facts (grounded, do not re-derive)

- Root archetype: `openEHR-EHR-COMPOSITION.encounter.v1` (PUBLISHED; local copy exists). Category = `event` (`openehr::433`).
- Narrative: `openEHR-EHR-EVALUATION.clinical_synopsis.v1` (PUBLISHED). `Synopsis` element path inside the EVALUATION = `/data[at0001]/items[at0002]` (DV_TEXT).
- Optional reason: `openEHR-EHR-EVALUATION.reason_for_encounter.v1` (PUBLISHED).
- Care setting code: *mental healthcare* = `openehr::802` (terminology group `setting`).
- Minted template `<id>` UUID (use exactly this): `7481293d-cea9-44b7-8cf8-e82f5fed4140`
- OET schema confirmed against a real CKM OET: `<definition xsi:type="COMPOSITION" archetype_id=… concept_name=…>` → `<Content xsi:type="EVALUATION" archetype_id=… concept_name=… path="/content">` → nested `<Rule path=… min=… max=… name=…>`; trailing `<Context/>`. `<integrity_checks>` digests are tool-generated and are **omitted** here (never hand-fabricated).

---

## Task 1: Author the GGZ Rapportage OET

**Files:**
- Create: `local/GGZ Rapportage.oet`

- [ ] **Step 1: Create the OET file**

Create `local/GGZ Rapportage.oet` with exactly this content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<template xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns="openEHR/v1/Template">
  <id>7481293d-cea9-44b7-8cf8-e82f5fed4140</id>
  <name>GGZ Rapportage</name>
  <description>
    <lifecycle_state>Initial</lifecycle_state>
    <details>
      <purpose>Vastleggen van een enkele GGZ-rapportage (decursus): het verslag van een behandelcontact. De rapportage is primair vrije tekst; SOEP (Subjectief/Objectief/Evaluatie/Plan) wordt in v1 als schrijfconventie (kopjes) gehanteerd. Optioneel kan de reden van contact worden vastgelegd.</purpose>
      <use>Per behandelcontact in de GGZ. De decursus is de chronologische reeks van deze composities en wordt opgevraagd via AQL (gesorteerd op context/start_time).</use>
      <misuse>Niet gebruiken voor gestructureerde SOEP-velden of koppeling aan behandeldoelen; die zijn voorzien voor v2.</misuse>
    </details>
    <other_details>
      <item>
        <key>Care setting</key>
        <value>mental healthcare (openEHR Terminology setting::802)</value>
      </item>
      <item>
        <key>Roadmap v2</key>
        <value>Coded contact type/modality; discrete SOEP (e.g. SECTION.soap.v1); behandeldoel linkage (EVALUATION.goal.v1 / RM LINK)</value>
      </item>
    </other_details>
  </description>
  <definition xsi:type="COMPOSITION" archetype_id="openEHR-EHR-COMPOSITION.encounter.v1" concept_name="GGZ Rapportage">
    <Content xsi:type="EVALUATION" archetype_id="openEHR-EHR-EVALUATION.clinical_synopsis.v1" concept_name="Clinical synopsis" path="/content" name="Rapportage" min="1" max="1">
      <Rule path="/data[at0001]/items[at0002]" min="1" max="1" name="Rapportage" />
    </Content>
    <Content xsi:type="EVALUATION" archetype_id="openEHR-EHR-EVALUATION.reason_for_encounter.v1" concept_name="Reason for encounter" path="/content" name="Reden van contact" min="0" max="1" />
    <Context />
  </definition>
</template>
```

- [ ] **Step 2: Verify the XML is well-formed**

Run:
```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
python3 -c "import xml.dom.minidom as m; m.parse('local/GGZ Rapportage.oet'); print('WELL-FORMED')"
```
Expected: prints `WELL-FORMED` with no traceback.

- [ ] **Step 3: Verify structural content with assertions**

Run:
```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
python3 - <<'PY'
import xml.etree.ElementTree as ET
ns = {'t': 'openEHR/v1/Template'}
root = ET.parse('local/GGZ Rapportage.oet').getroot()
assert root.find('t:name', ns).text == 'GGZ Rapportage', 'name mismatch'
d = root.find('t:definition', ns)
assert d.get('archetype_id') == 'openEHR-EHR-COMPOSITION.encounter.v1', 'root archetype mismatch'
contents = d.findall('t:Content', ns)
ids = [c.get('archetype_id') for c in contents]
assert 'openEHR-EHR-EVALUATION.clinical_synopsis.v1' in ids, 'missing clinical_synopsis'
assert 'openEHR-EHR-EVALUATION.reason_for_encounter.v1' in ids, 'missing reason_for_encounter'
cs = [c for c in contents if c.get('archetype_id').endswith('clinical_synopsis.v1')][0]
assert cs.get('min') == '1' and cs.get('max') == '1', 'clinical_synopsis must be 1..1'
rfe = [c for c in contents if c.get('archetype_id').endswith('reason_for_encounter.v1')][0]
assert rfe.get('min') == '0' and rfe.get('max') == '1', 'reason_for_encounter must be 0..1'
print('STRUCTURE OK:', ids)
PY
```
Expected: prints `STRUCTURE OK: ['openEHR-EHR-EVALUATION.clinical_synopsis.v1', 'openEHR-EHR-EVALUATION.reason_for_encounter.v1']` and no AssertionError.

- [ ] **Step 4: Verify semantics with the template-explain skill**

Invoke the skill on the new file:
```
/template-explain local/GGZ Rapportage.oet
```
Expected: it parses the OET and reports a COMPOSITION (Encounter) root named "GGZ Rapportage" with two EVALUATION contents — "Rapportage" (clinical_synopsis, mandatory, single) and "Reden van contact" (reason_for_encounter, optional). If `/template-explain` reports a schema error on any attribute (e.g. `min` on `Content`), correct only the flagged attribute (e.g. drop `min`, keep `max="1"`), then re-run Steps 2–4.

- [ ] **Step 5: Commit**

```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
git add "local/GGZ Rapportage.oet"
git commit -m "$(cat <<'EOF'
Add GGZ Rapportage OET template

GGZ decursus contact report: COMPOSITION.encounter (event) with a
mandatory EVALUATION.clinical_synopsis (Rapportage) and optional
EVALUATION.reason_for_encounter. Pure reuse, zero new archetypes.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
EOF
)"
```
Expected: one new file committed.

---

## Task 2 (OPTIONAL validation appendix): demonstrate the decursus query

> Scope note: the user narrowed v1 to "just the OET file". This task only
> *demonstrates* spec success criteria #2 and #3; it produces no archetype/template
> artifacts. Skip if not wanted.

**Files:**
- Create: `docs/superpowers/examples/ggz-rapportage-decursus.aql`

- [ ] **Step 1: Design the decursus AQL**

Invoke:
```
/aql-designer retrieve a patient's GGZ decursus: all encounter compositions
containing a clinical_synopsis, returning composition start_time and the
Synopsis text, ordered chronologically by context/start_time
```
Expected: an AQL roughly of this shape (the designer is the source of truth — use its output):
```sql
SELECT c/context/start_time/value AS contact_time,
       e/data[at0001]/items[at0002]/value/value AS rapportage
FROM EHR e_ehr
  CONTAINS COMPOSITION c[openEHR-EHR-COMPOSITION.encounter.v1]
    CONTAINS EVALUATION e[openEHR-EHR-EVALUATION.clinical_synopsis.v1]
WHERE e_ehr/ehr_id/value = $ehrId
ORDER BY c/context/start_time/value ASC
```

- [ ] **Step 2: Save the AQL**

Write the designer's validated AQL to `docs/superpowers/examples/ggz-rapportage-decursus.aql`.

- [ ] **Step 3: Sanity-check a FLAT instance shape (no file required)**

Invoke:
```
/format-data design a FLAT composition for the GGZ Rapportage template: one
contact with start_time, composer/participation as behandelaar, a Synopsis
narrative, and a reden van contact
```
Expected: a FLAT JSON sketch with keys for `ggz_rapportage/context/start_time`, the clinical_synopsis Synopsis path, and an optional reason_for_encounter path. This is a design check only — confirm the paths line up with the OET; do not commit a file unless desired.

- [ ] **Step 4: Commit (only if Step 2 produced a file)**

```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
git add "docs/superpowers/examples/ggz-rapportage-decursus.aql"
git commit -m "$(cat <<'EOF'
Add GGZ Rapportage decursus AQL example

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-review checklist (completed by plan author)

- **Spec coverage:** Root encounter/event ✓ (Task 1 Step 1, definition); clinical_synopsis mandatory narrative ✓ (Content min/max + Synopsis Rule); reason_for_encounter optional ✓; setting documented ✓ (other_details); reporter via RM ✓ (no archetype, nothing to author — RM-only, noted in spec §5); zero new archetypes ✓; OET-only deliverable ✓ (Decision B revised); v2 items excluded ✓ (Roadmap note). Success criteria #1 ✓ (Steps 2–4), #2/#3 ✓ (Task 2).
- **Placeholder scan:** None. The UUID is concrete; `<integrity_checks>` are deliberately omitted (tool-generated), not a gap.
- **Type/path consistency:** Synopsis path `/data[at0001]/items[at0002]` used consistently in OET, AQL, and assertions; archetype IDs identical across all tasks.
