# Six-Minute Walk Test (6MWT) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Author a new `OBSERVATION.six_minute_walk_test.v0` archetype and a `report-result`-rooted web template that composes it with reused vital-sign observations.

**Architecture:** One new ADL 1.4 OBSERVATION archetype (single summary event; embedded Borg CR10 ordinals; supplemental-O₂ slot to `CLUSTER.inspired_oxygen.v1`) plus one new Archetype-Designer CAM `.t.json` template rooting `COMPOSITION.report-result.v1` and composing `pulse.v2` / `blood_pressure.v2` / `pulse_oximetry.v1`. Reused archetypes are referenced by CKM id, never vendored into `local/`.

**Tech Stack:** ADL 1.4; openEHR RM 1.1.0; openEHR Web Template (CAM differential) JSON; openehr-assistant MCP tools (`archetype-authoring`, `archetype-lint`, `ckm_archetype_get`, `guide_adl_idiom_lookup`, `terminology_resolve`).

**Source spec:** `docs/superpowers/specs/2026-06-09-six-minute-walk-test-design.md` — read it before starting.

---

## File Structure

| File | Responsibility | Action |
|---|---|---|
| `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl` | The 6MWT observation: distance, summary physiology, Borg, supplemental-O₂ slot | Create |
| `local/Six minute walk test.t.json` | Template rooting `report-result.v1`, composing the 6MWT + 3 vitals | Create |
| `docs/superpowers/plans/2026-06-09-six-minute-walk-test.md` | This plan | (exists) |

No existing files are modified. No reused CKM archetype is copied locally.

---

## Reference values (used by multiple tasks)

**Archetype header (line 1–2):**
- `archetype (adl_version=1.4; uid=<GENERATE-FRESH-UUID>)`
- id line 2: `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0`
- `concept = [at0000]`; `original_language = <[ISO_639-1::en]>`

**at-code → node blueprint (data event tree):**

| at-code | RM type | Label | Occurrences | Constraint |
|---|---|---|---|---|
| at0000 | OBSERVATION | Six-minute walk test | — | root |
| at0001 | HISTORY | History | — | `data` |
| at0002 | EVENT | 6MWT summary | 1..1 | single event |
| at0003 | ITEM_TREE | Tree | — | event `data` |
| at0004 | ELEMENT→DV_QUANTITY | Distance walked | 1..1 | property=length; units `m`, magnitude ≥ 0; 0 dec |
| at0005 | ELEMENT→DV_COUNT | Laps completed | 0..1 | magnitude ≥ 0 |
| at0006 | ELEMENT→DV_QUANTITY | Lowest SpO₂ during test | 0..1 | units `%`, 0..100, 0 dec |
| at0007 | ELEMENT→DV_QUANTITY | Peak heart rate during test | 0..1 | property=rate; units `/min`, ≥ 0, 0 dec |
| at0008 | ELEMENT→DV_ORDINAL | Borg dyspnoea (end of test) | 0..1 | Borg CR10 set (see Task 3) |
| at0009 | ELEMENT→DV_ORDINAL | Borg fatigue (end of test) | 0..1 | Borg CR10 set (see Task 3) |
| at0010 | ELEMENT→DV_BOOLEAN | Test completed | 0..1 | — |
| at0011 | ELEMENT→DV_TEXT | Clinical interpretation | 0..1 | — |
| at0012 | ELEMENT→DV_TEXT | Comment | 0..1 | — |
| at0013 | ITEM_TREE | Tree | — | event `state` |
| at0014 | ARCHETYPE_SLOT | Supplemental oxygen | 0..1 | include `CLUSTER.inspired_oxygen\.v1` |
| at0015 | ITEM_TREE | Tree | — | `protocol` |
| at0016 | ARCHETYPE_SLOT | Extension | 0..* | include any `CLUSTER` |

**Borg CR10 ordinal labels (Task 3):** 0 Nothing at all · 1 Very slight · 2 Slight · 3 Moderate · 4 Somewhat severe · 5 Severe · 6 (—) · 7 Very severe · 8 (—) · 9 Very, very severe (almost maximal) · 10 Maximal.

**Reused ids (template, Task 7):** `openEHR-EHR-COMPOSITION.report-result.v1` (root) · `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0` · `openEHR-EHR-OBSERVATION.pulse.v2` · `openEHR-EHR-OBSERVATION.blood_pressure.v2` · `openEHR-EHR-OBSERVATION.pulse_oximetry.v1`.

---

## Task 1: Fetch the structural precedent & scaffold the archetype header

**Files:**
- Create: `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

- [ ] **Step 1: Invoke the authoring skill.** Run the `archetype-authoring` skill (it owns the create lifecycle and ADL 1.4 idioms). State the goal: author `OBSERVATION.six_minute_walk_test.v0` per the spec.

- [ ] **Step 2: Fetch the precedent ADL** to copy the exact ADL 1.4 OBSERVATION idiom (HISTORY/EVENT/ITEM_TREE skeleton, ontology block shape):

```
mcp__openehr-assistant__ckm_archetype_get(identifier="openEHR-EHR-OBSERVATION.timed_25_foot_walk.v1", format="adl")
```

Read its `definition` and `ontology` sections — do NOT copy its concept/at-codes, only the structural idiom.

- [ ] **Step 3: Generate a fresh UUID** for the header `uid`:

```bash
uuidgen | tr 'A-Z' 'a-z'
```

- [ ] **Step 4: Write the header + metadata sections** (`archetype` … `concept` … `language` … `description`). Match the section order and tab indentation of an existing local OBSERVATION (e.g. `local/openEHR-EHR-OBSERVATION.exam.v1.adl`). Use the UUID from Step 3, `original_language = <[ISO_639-1::en]>`, author = the workspace owner, `lifecycle_state = <"unmanaged">` or `<"0">` consistent with local drafts. Concept text = "Six-minute walk test".

- [ ] **Step 5: Verify the file parses as far as the header.** Run the lint to confirm no fatal parse error in the metadata (definition will still be incomplete — ERRORs about missing definition are expected here):

```
/archetype-lint local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl
```
Expected: parses; reports missing/empty `definition` — that is fine at this step.

- [ ] **Step 6: Commit**

```bash
git add "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
git commit -m "feat(6mwt): scaffold six_minute_walk_test.v0 header and metadata"
```

---

## Task 2: Author the definition tree (data event + elements + state slot)

**Files:**
- Modify: `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

- [ ] **Step 1: Write the `definition` block** following the blueprint table above. Structure:

```
definition
    OBSERVATION[at0000] matches {
        data matches {
            HISTORY[at0001] matches {
                events cardinality matches {1..*; unordered} matches {
                    EVENT[at0002] occurrences matches {1..1} matches {
                        data matches {
                            ITEM_TREE[at0003] matches {
                                items cardinality matches {0..*; unordered} matches {
                                    ELEMENT[at0004] occurrences matches {1..1} matches { value matches { C_DV_QUANTITY <property=[openehr::122]; list = <["1"]={units=<"m">; magnitude=<|>=0.0|>; precision=<|0|>}>> } }
                                    ELEMENT[at0005] occurrences matches {0..1} matches { value matches { DV_COUNT matches { magnitude matches {|>=0|} } } }
                                    ELEMENT[at0006] occurrences matches {0..1} matches { value matches { C_DV_QUANTITY <property=[openehr::125]; list=<["1"]={units=<"%">; magnitude=<|0.0..100.0|>; precision=<|0|>}>> } }
                                    ELEMENT[at0007] occurrences matches {0..1} matches { value matches { C_DV_QUANTITY <property=[openehr::384]; list=<["1"]={units=<"/min">; magnitude=<|>=0.0|>; precision=<|0|>}>> } }
                                    ELEMENT[at0008] occurrences matches {0..1} matches { value matches { /* Borg CR10 ordinal — Task 3 */ } }
                                    ELEMENT[at0009] occurrences matches {0..1} matches { value matches { /* Borg CR10 ordinal — Task 3 */ } }
                                    ELEMENT[at0010] occurrences matches {0..1} matches { value matches { DV_BOOLEAN matches {value matches {True, False}} } }
                                    ELEMENT[at0011] occurrences matches {0..1} matches { value matches { DV_TEXT matches {*} } }
                                    ELEMENT[at0012] occurrences matches {0..1} matches { value matches { DV_TEXT matches {*} } }
                                }
                            }
                        }
                        state matches {
                            ITEM_TREE[at0013] matches {
                                items cardinality matches {0..*; unordered} matches {
                                    allow_archetype CLUSTER[at0014] occurrences matches {0..1} matches {
                                        include archetype_id/value matches {/openEHR-EHR-CLUSTER\.inspired_oxygen\.v1/}
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
```

Confirm the exact `property` codes with the idiom helper before finalizing (length, oxygen-saturation, heart-rate properties):

```
mcp__openehr-assistant__guide_adl_idiom_lookup(...)   # C_DV_QUANTITY property/units idiom
mcp__openehr-assistant__terminology_resolve(...)      # confirm openehr property codes if unsure
```

- [ ] **Step 2: Add the `protocol` branch** to the `OBSERVATION[at0000]` block (sibling of `data`):

```
        protocol matches {
            ITEM_TREE[at0015] matches {
                items cardinality matches {0..*; unordered} matches {
                    allow_archetype CLUSTER[at0016] occurrences matches {0..*} matches {
                        include archetype_id/value matches {/.*/}
                    }
                }
            }
        }
```

- [ ] **Step 3: Lint (expect ontology ERRORs only).**

```
/archetype-lint local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl
```
Expected: definition parses; remaining ERRORs are "at-code not defined in terminology" for at0000–at0016 — fixed in Task 5. No structural/cardinality ERRORs.

- [ ] **Step 4: Commit**

```bash
git add "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
git commit -m "feat(6mwt): add definition tree, supplemental-O2 slot, extension slot"
```

---

## Task 3: Author the Borg CR10 ordinals (at0008, at0009)

**Files:**
- Modify: `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

- [ ] **Step 1: Define the ordinal for `at0008` (Borg dyspnoea)** with 11 values 0–10, each pointing at a dedicated local symbol code. Allocate codes at0018–at0028:

```
value matches {
    DV_ORDINAL matches {
        value matches {
            0|[at0018], 1|[at0019], 2|[at0020], 3|[at0021], 4|[at0022],
            5|[at0023], 6|[at0024], 7|[at0025], 8|[at0026], 9|[at0027], 10|[at0028]
        }
    }
}
```

- [ ] **Step 2: Define the ordinal for `at0009` (Borg fatigue)** with its own symbol codes at0029–at0039 (same 0–10 labels; ADL 1.4 requires distinct codes per ordinal element):

```
value matches {
    DV_ORDINAL matches {
        value matches {
            0|[at0029], 1|[at0030], 2|[at0031], 3|[at0032], 4|[at0033],
            5|[at0034], 6|[at0035], 7|[at0036], 8|[at0037], 9|[at0038], 10|[at0039]
        }
    }
}
```

- [ ] **Step 3: Splice these two `value` blocks** into the `ELEMENT[at0008]` and `ELEMENT[at0009]` placeholders left in Task 2, Step 1.

- [ ] **Step 4: Commit** (ontology entries for the new codes come in Task 5):

```bash
git add "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
git commit -m "feat(6mwt): add Borg CR10 dyspnoea and fatigue ordinals"
```

---

## Task 4: Author the ontology / terminology block

**Files:**
- Modify: `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

- [ ] **Step 1: Write the `ontology` section** with `term_definitions = <["en"] = <items = <…>>>` for every node and ordinal code. Use these exact `text` values (add a one-line `description` for each; for ordinal symbol codes, `description` may repeat the `text`):

| code | text |
|---|---|
| at0000 | Six-minute walk test |
| at0001 | History |
| at0002 | 6MWT summary |
| at0003 | Tree |
| at0004 | Distance walked |
| at0005 | Laps completed |
| at0006 | Lowest SpO₂ during test |
| at0007 | Peak heart rate during test |
| at0008 | Borg dyspnoea (end of test) |
| at0009 | Borg fatigue (end of test) |
| at0010 | Test completed |
| at0011 | Clinical interpretation |
| at0012 | Comment |
| at0013 | Tree |
| at0014 | Supplemental oxygen |
| at0015 | Tree |
| at0016 | Extension |
| at0018–at0028 | Borg dyspnoea levels 0–10: Nothing at all / Very slight / Slight / Moderate / Somewhat severe / Severe / (level 6) / Very severe / (level 8) / Very, very severe (almost maximal) / Maximal |
| at0029–at0039 | Borg fatigue levels 0–10: (same 11 labels as at0018–at0028) |

- [ ] **Step 2: Add descriptions** for the key clinical elements (longer, clinician-facing). Minimum, use:
  - at0004: "The total distance walked during the six-minute walk test (6MWD)."
  - at0006: "The lowest peripheral oxygen saturation (SpO₂) observed during the test."
  - at0007: "The highest heart rate observed during the test."
  - at0008: "Patient-rated breathlessness at the end of the test on the modified Borg CR10 scale."
  - at0009: "Patient-rated leg/general fatigue at the end of the test on the modified Borg CR10 scale."
  - at0014: "Details of any supplemental oxygen delivered during the test."
  - at0016: "Additional structured detail not otherwise modelled."

- [ ] **Step 3: Run STRICT lint and confirm zero ERRORs.**

```
/archetype-lint local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl strict
```
Expected: 0 ERRORs. WARNINGs/INFO acceptable; review each and resolve or justify in the description.

- [ ] **Step 4: Commit**

```bash
git add "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
git commit -m "feat(6mwt): add ontology term definitions; passes STRICT lint"
```

---

## Task 5: Remediate lint findings to zero ERRORs

**Files:**
- Modify: `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

- [ ] **Step 1: If Task 4 Step 3 reported any ERROR**, invoke `/archetype-fix-syntax` (or the `archetype-authoring` remediation path) and apply fixes. Common items: missing `description` on a code, slot regex anchoring, `C_DV_QUANTITY` property code mismatch.

- [ ] **Step 2: Re-lint STRICT to confirm clean.**

```
/archetype-lint local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl strict
```
Expected: 0 ERRORs.

- [ ] **Step 3: Commit (only if changes were made)**

```bash
git add "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
git commit -m "fix(6mwt): remediate lint findings to zero errors"
```

---

## Task 6: Scaffold the template from the Vital signs precedent

**Files:**
- Create: `local/Six minute walk test.t.json`

- [ ] **Step 1: Invoke the `template-authoring` skill.** Goal: a CAM differential `.t.json` mirroring `local/Vital signs.t.json`, rooting `report-result.v1`.

- [ ] **Step 2: Copy the top-level scaffold** from `local/Vital signs.t.json` and edit these keys:
  - `templateId`: `"Six minute walk test"`
  - `parentArchetypeId`: `"openEHR-EHR-COMPOSITION.report-result.v1"`
  - `archetypeId.value`: `"openEHR-EHR-COMPOSITION.t_six_minute_walk_test.v1"`
  - `description.details.en.purpose`: "A template for recording a six-minute walk test (6MWT): the walk summary plus heart rate, blood pressure and pulse oximetry measured at rest and after the test."
  - Drop the non-`en` `details`/`translations` entries (this is a new en-only artifact); keep the `en` block.
  - `uid`: generate fresh via `uuidgen`.
  - `otherDetails.MD5-CAM-1.0.1` → `"REGENERATE_IN_ARCHETYPE_DESIGNER"`; `otherDetails.PARENT:MD5-CAM-1.0.1` → `"REGENERATE_IN_ARCHETYPE_DESIGNER"`; `buildUid` → `"REGENERATE_IN_ARCHETYPE_DESIGNER"`.

- [ ] **Step 3: Validate JSON parses.**

```bash
python3 -m json.tool "local/Six minute walk test.t.json" > /dev/null && echo VALID_JSON
```
Expected: `VALID_JSON`.

- [ ] **Step 4: Commit**

```bash
git add "local/Six minute walk test.t.json"
git commit -m "feat(6mwt): scaffold report-result-rooted template"
```

---

## Task 7: Populate the template `definition` with the four content archetypes

**Files:**
- Modify: `local/Six minute walk test.t.json`

- [ ] **Step 1: Build the `definition` tree** as a `C_COMPLEX_OBJECT` of `rmTypeName: "COMPOSITION"` with a `context` attribute (EVENT_CONTEXT, as in Vital signs) **and** a `content` `C_ATTRIBUTE` whose `children` are four `C_ARCHETYPE_ROOT` nodes (one per archetype id below), with `occurrences`:

| archetypeId | occurrences |
|---|---|
| `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0` | `{"lowerIncluded":true,"lower":1,"upperIncluded":true,"upper":1}` |
| `openEHR-EHR-OBSERVATION.pulse.v2` | `0..1` |
| `openEHR-EHR-OBSERVATION.blood_pressure.v2` | `0..1` |
| `openEHR-EHR-OBSERVATION.pulse_oximetry.v1` | `0..1` |

Follow the `C_ARCHETYPE_ROOT` shape used by Archetype Designer (`"@type":"C_ARCHETYPE_ROOT"`, `archetypeId.value`, `nodeId` = the archetype's own `at0000`). If unsure of the exact node shape, fetch a multi-archetype template example:

```
mcp__openehr-assistant__examples_search(query="web template composition content archetype_root")
```

- [ ] **Step 2: Add narrowing notes** in each content node where the source archetype offers non-metric units — constrain to metric only (distance `m`, SpO₂ `%`, HR `/min`, BP `mmHg`), mirroring the Vital signs template's stated approach.

- [ ] **Step 3: Validate JSON parses.**

```bash
python3 -m json.tool "local/Six minute walk test.t.json" > /dev/null && echo VALID_JSON
```
Expected: `VALID_JSON`.

- [ ] **Step 4: Commit**

```bash
git add "local/Six minute walk test.t.json"
git commit -m "feat(6mwt): compose 6MWT + pulse/BP/SpO2 into template content"
```

---

## Task 8: Cross-reference validation & final review

**Files:** none modified (verification only)

- [ ] **Step 1: Confirm every template archetype id resolves.** The new OBSERVATION must exist locally; the three vitals + composition root are reused CKM ids (referenced, not vendored):

```bash
ls "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
grep -o 'openEHR-EHR-[A-Za-z]*\.[a-z_0-9-]*\.v[0-9]' "local/Six minute walk test.t.json" | sort -u   # hyphen-aware: matches report-result
```
Expected: the new OBSERVATION file exists; grep lists exactly the five ids from the Reference values section.

- [ ] **Step 2: Run the impact command** to confirm the template references the new archetype as expected:

```
/archetype-impact openEHR-EHR-OBSERVATION.six_minute_walk_test.v0
```
Expected: the impact table lists `Six minute walk test.t.json`.

- [ ] **Step 3: Final STRICT lint of the archetype.**

```
/archetype-lint local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl strict
```
Expected: 0 ERRORs.

- [ ] **Step 4: Note the hash-regeneration follow-up.** The template's `MD5-CAM` / `buildUid` are placeholders; record in the PR/commit description that the template must be re-exported from Archetype Designer before production use.

- [ ] **Step 5: Final commit (if any cleanup made)**

```bash
git add -A
git commit -m "chore(6mwt): cross-reference validation and notes"
```

---

## Self-Review (completed by plan author)

- **Spec coverage:** OBSERVATION structure (Task 2/3), Borg embedding (Task 3), supplemental-O₂ slot (Task 2), report-result root + composition (Task 6/7), reuse-by-reference (Task 8 Step 1), hashes flagged (Task 6 Step 2 / Task 8 Step 4), acceptance criteria (Task 4/5/8). Deferred items remain deferred — no tasks, by design.
- **Placeholder scan:** the only literal placeholders are the deliberate `REGENERATE_IN_ARCHETYPE_DESIGNER` hash markers and the `<GENERATE-FRESH-UUID>` instruction — both are explicit, actionable instructions, not unspecified work.
- **Consistency:** at-codes at0000–at0016 (nodes) and at0018–at0039 (ordinals) are allocated once and reused consistently; the five archetype ids are identical across Tasks 7 and 8.
