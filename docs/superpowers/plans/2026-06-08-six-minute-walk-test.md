# 6-Minute Walk Test (6MWT) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a minimal, distance-focused 6-Minute Walk Test artifact set — one new OBSERVATION archetype plus one template — to the `local/` content of this openEHR sandbox.

**Architecture:** Author a single self-contained `OBSERVATION.six_minute_walk_test.v0` archetype (ADL 1.4) holding distance walked, duration, laps, an inline Borg breathlessness ordinal, walking-aid coded text, course length and a comment — no nested CLUSTERs, no external slots. Then bind it into a `6 Minute Walk Test` template rooted on the existing `COMPOSITION.encounter.v1`. "Tests" in this content repo are `/archetype-lint` (22 normative rules) plus structural/referential checks, not unit tests.

**Tech Stack:** ADL 1.4 (`adl_version=1.4`), openEHR RM (EHR), openEHR Web Template JSON, the `openehr-assistant` MCP tools / workspace slash commands (`/archetype-lint`, `/template-explain`), `jq`, `uuidgen`.

**Spec:** `docs/superpowers/specs/2026-06-08-six-minute-walk-test-design.md`

**Conventions grounded from this repo / MCP (do not re-derive):**
- ADL 1.4 file/section order and `C_DV_QUANTITY <...>` idiom follow `local/openEHR-EHR-OBSERVATION.ecg_result.v1.adl`.
- Coded value sets are written **inline** in `defining_code` as a `[local:: atNNNN, ...]` list (as `ecg_result` does), **not** as separate `value_sets`/ac-codes — no local archetype here uses a `value_sets` block.
- Borg uses the ADL 1.4 inline ordinal idiom: `value matches { 0|[local::at0013], 1|[local::at0014], ... }`.
- openEHR property code for length (units `m`) is `[openehr::122]` (resolved via `terminology_resolve`).
- Templates carry tool-generated `uid`/`buildUid`/`MD5-CAM` hashes; per repo CLAUDE.md these MUST NOT be hand-fabricated. The `uid` is a fresh UUID (OK to generate with `uuidgen`); MD5/buildUid are produced by an OPT compiler / Archetype Designer.

---

## File Structure

- **Create:** `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl` — the new OBSERVATION archetype (the core intellectual artifact; hand-authored, lint-clean).
- **Create:** `local/6 Minute Walk Test.t.json` — the template binding the new OBSERVATION into a `COMPOSITION.encounter.v1`.
- **Reference only (unchanged):** `local/openEHR-EHR-COMPOSITION.encounter.v1.adl` (template root), `local/openEHR-EHR-OBSERVATION.ecg_result.v1.adl` (style reference), `local/Vital signs.t.json` (template shape reference).

at-code allocation (single source of truth — Task 1 and Task 2 must agree):

| Code | Rubric | RM / type | Section |
|---|---|---|---|
| at0000 | 6 Minute Walk Test | OBSERVATION | root |
| at0001 | History | HISTORY | data (internal) |
| at0002 | End of test | POINT_EVENT | event |
| at0003 | Tree | ITEM_TREE | data (internal) |
| at0004 | Distance walked | ELEMENT / DV_QUANTITY (m) | data |
| at0005 | Duration walked | ELEMENT / DV_DURATION | data |
| at0006 | Number of laps | ELEMENT / DV_COUNT | data |
| at0007 | Borg breathlessness score | ELEMENT / DV_ORDINAL | data |
| at0008 | Tree | ITEM_TREE | state (internal) |
| at0009 | Walking aid used | ELEMENT / DV_CODED_TEXT | state |
| at0010 | Tree | ITEM_TREE | protocol (internal) |
| at0011 | Course length | ELEMENT / DV_QUANTITY (m) | protocol |
| at0012 | Comment | ELEMENT / DV_TEXT | protocol |
| at0013–at0023 | Borg ordinal 0–10 | ordinal codes | data |
| at0024–at0029 | None / cane / crutches / frame / wheelchair / other | walking-aid codes | state |

---

## Task 1: Author the OBSERVATION archetype

**Files:**
- Create: `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`

- [ ] **Step 1: Generate a fresh UID for the archetype header**

Run: `uuidgen`
Expected: a UUID like `3f2b1c4a-...`. Copy it; substitute it for `<GENERATED-UID>` in Step 2.

- [ ] **Step 2: Write the archetype file**

Create `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl` with EXACTLY this content (tabs for indentation; replace `<GENERATED-UID>` with the Step 1 value):

```adl
archetype (adl_version=1.4; uid=<GENERATED-UID>)
	openEHR-EHR-OBSERVATION.six_minute_walk_test.v0

concept
	[at0000]

language
	original_language = <[ISO_639-1::en]>

description
	original_author = <
		["date"] = <"2026-06-08">
		["name"] = <"Sebastian Iancu">
		["organisation"] = <"Code24">
		["email"] = <"sebastian.iancu@code24.nl">
	>
	lifecycle_state = <"unmanaged">
	details = <
		["en"] = <
			language = <[ISO_639-1::en]>
			purpose = <"To record the result of a 6-minute walk test (6MWT): a sub-maximal test of functional exercise capacity in which a person walks as far as possible on a flat, hard surface in 6 minutes.">
			keywords = <"6MWT", "six minute walk", "walk test", "functional exercise capacity", "walking distance", "exercise tolerance">
			use = <"Use to record the primary outcome of a 6-minute walk test - the total distance walked - together with the walking duration achieved, number of laps, an optional Borg breathlessness rating, the walking aid used, the course length and a free-text comment.

This is a minimal, distance-focused model. Paired pre/post vital signs (pulse oximetry, pulse, blood pressure), supplemental oxygen, rest/stop episodes and an overall clinical interpretation are intentionally out of scope and may be added in a template or a later version by composing the relevant published archetypes.">
			misuse = <"Not to be used to record other timed or distance walk tests, such as the timed 25-foot walk, the incremental shuttle walk test or the 2-minute walk test.

Not to be used to record continuous gait analysis or step counts from activity trackers.">
			copyright = <"© openEHR Foundation">
		>
	>
	other_details = <
		["licence"] = <"This work is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/.">
		["custodian_organisation"] = <"openEHR Foundation">
		["original_namespace"] = <"org.openehr">
		["original_publisher"] = <"openEHR Foundation">
		["custodian_namespace"] = <"org.openehr">
	>

definition
	OBSERVATION[at0000] matches {    -- 6 Minute Walk Test
		data matches {
			HISTORY[at0001] matches {    -- History
				events cardinality matches {1..*; unordered} matches {
					POINT_EVENT[at0002] occurrences matches {1..1} matches {    -- End of test
						data matches {
							ITEM_TREE[at0003] matches {    -- Tree
								items cardinality matches {0..*; unordered} matches {
									ELEMENT[at0004] occurrences matches {1..1} matches {    -- Distance walked
										value matches {
											C_DV_QUANTITY <
												property = <[openehr::122]>
												list = <
													["1"] = <
														units = <"m">
														magnitude = <|>=0.0|>
														precision = <|0|>
													>
												>
											>
										}
									}
									ELEMENT[at0005] occurrences matches {0..1} matches {    -- Duration walked
										value matches {
											DV_DURATION matches {
												value matches {|>=PT0S|}
											}
										}
									}
									ELEMENT[at0006] occurrences matches {0..1} matches {    -- Number of laps
										value matches {
											DV_COUNT matches {
												magnitude matches {|>=0|}
											}
										}
									}
									ELEMENT[at0007] occurrences matches {0..1} matches {    -- Borg breathlessness score
										value matches {
											0|[local::at0013],    -- Nothing at all
											1|[local::at0014],    -- Very slight
											2|[local::at0015],    -- Slight
											3|[local::at0016],    -- Moderate
											4|[local::at0017],    -- Somewhat severe
											5|[local::at0018],    -- Severe
											6|[local::at0019],    -- Severe to very severe
											7|[local::at0020],    -- Very severe
											8|[local::at0021],    -- Very severe to very, very severe
											9|[local::at0022],    -- Very, very severe
											10|[local::at0023]    -- Maximal
										}
									}
								}
							}
						}
						state matches {
							ITEM_TREE[at0008] matches {    -- Tree
								items cardinality matches {0..*; unordered} matches {
									ELEMENT[at0009] occurrences matches {0..1} matches {    -- Walking aid used
										value matches {
											DV_CODED_TEXT matches {
												defining_code matches {
													[local::
													at0024,    -- None
													at0025,    -- Walking stick/cane
													at0026,    -- Crutches
													at0027,    -- Walking frame/rollator
													at0028,    -- Wheelchair
													at0029]    -- Other
												}
											}
										}
									}
								}
							}
						}
					}
				}
			}
		}
		protocol matches {
			ITEM_TREE[at0010] matches {    -- Tree
				items cardinality matches {0..*; unordered} matches {
					ELEMENT[at0011] occurrences matches {0..1} matches {    -- Course length
						value matches {
							C_DV_QUANTITY <
								property = <[openehr::122]>
								list = <
									["1"] = <
										units = <"m">
										magnitude = <|>=0.0|>
										precision = <|0|>
									>
								>
							>
						}
					}
					ELEMENT[at0012] occurrences matches {0..1} matches {    -- Comment
						value matches {
							DV_TEXT matches {*}
						}
					}
				}
			}
		}
	}

ontology
	term_definitions = <
		["en"] = <
			items = <
				["at0000"] = <
					text = <"6 Minute Walk Test">
					description = <"The result of a 6-minute walk test (6MWT), a sub-maximal test of functional exercise capacity.">
				>
				["at0001"] = <
					text = <"History">
					description = <"@ internal @">
				>
				["at0002"] = <
					text = <"End of test">
					description = <"The summary measurements recorded at the completion of the 6-minute walk test.">
				>
				["at0003"] = <
					text = <"Tree">
					description = <"@ internal @">
				>
				["at0004"] = <
					text = <"Distance walked">
					description = <"Total distance walked during the 6-minute walk test.">
					comment = <"The primary outcome of the test.">
				>
				["at0005"] = <
					text = <"Duration walked">
					description = <"The actual walking time achieved during the test.">
					comment = <"Nominally 6 minutes (PT6M); record the actual duration if the test was stopped early.">
				>
				["at0006"] = <
					text = <"Number of laps">
					description = <"The number of complete laps of the course walked during the test.">
				>
				["at0007"] = <
					text = <"Borg breathlessness score">
					description = <"Rating of perceived breathlessness using the modified Borg CR10 scale.">
					comment = <"Integer subset (0-10) of the modified Borg CR10 scale; the canonical 0.5 step is not represented because DV_ORDINAL requires integer values.">
				>
				["at0008"] = <
					text = <"Tree">
					description = <"@ internal @">
				>
				["at0009"] = <
					text = <"Walking aid used">
					description = <"The walking aid, if any, used by the person during the test.">
				>
				["at0010"] = <
					text = <"Tree">
					description = <"@ internal @">
				>
				["at0011"] = <
					text = <"Course length">
					description = <"The length of the walking course (e.g. the straight track length between turnaround points).">
				>
				["at0012"] = <
					text = <"Comment">
					description = <"Additional narrative about the 6-minute walk test result, not captured in other fields.">
				>
				["at0013"] = <
					text = <"Nothing at all">
					description = <"No breathlessness at all (Borg CR10 = 0).">
				>
				["at0014"] = <
					text = <"Very slight">
					description = <"Very slight breathlessness (Borg CR10 = 1).">
				>
				["at0015"] = <
					text = <"Slight">
					description = <"Slight (light) breathlessness (Borg CR10 = 2).">
				>
				["at0016"] = <
					text = <"Moderate">
					description = <"Moderate breathlessness (Borg CR10 = 3).">
				>
				["at0017"] = <
					text = <"Somewhat severe">
					description = <"Somewhat severe breathlessness (Borg CR10 = 4).">
				>
				["at0018"] = <
					text = <"Severe">
					description = <"Severe (heavy) breathlessness (Borg CR10 = 5).">
				>
				["at0019"] = <
					text = <"Severe to very severe">
					description = <"Breathlessness between severe and very severe (Borg CR10 = 6).">
				>
				["at0020"] = <
					text = <"Very severe">
					description = <"Very severe breathlessness (Borg CR10 = 7).">
				>
				["at0021"] = <
					text = <"Very severe to very, very severe">
					description = <"Breathlessness between very severe and very, very severe (Borg CR10 = 8).">
				>
				["at0022"] = <
					text = <"Very, very severe">
					description = <"Very, very severe (almost maximal) breathlessness (Borg CR10 = 9).">
				>
				["at0023"] = <
					text = <"Maximal">
					description = <"Maximal breathlessness (Borg CR10 = 10).">
				>
				["at0024"] = <
					text = <"None">
					description = <"No walking aid was used.">
				>
				["at0025"] = <
					text = <"Walking stick/cane">
					description = <"A walking stick or cane was used.">
				>
				["at0026"] = <
					text = <"Crutches">
					description = <"One or more crutches were used.">
				>
				["at0027"] = <
					text = <"Walking frame/rollator">
					description = <"A walking frame or rollator was used.">
				>
				["at0028"] = <
					text = <"Wheelchair">
					description = <"A wheelchair was used (self-propelled or assisted).">
				>
				["at0029"] = <
					text = <"Other">
					description = <"Another walking aid not otherwise listed was used.">
				>
			>
		>
	>
```

- [ ] **Step 3: Sanity-check structure before linting**

Run:
```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
head -2 "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
grep -c "at00" "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
```
Expected: line 2 is exactly `\topenEHR-EHR-OBSERVATION.six_minute_walk_test.v0` (matches the filename), and the at-code count is non-zero. Confirm every `atNNNN` used in `definition` (at0000–at0029) has a matching `term_definitions` entry by eye against the table above.

- [ ] **Step 4: Lint the archetype (this is the "test")**

Invoke the workspace command `/archetype-lint` on `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl` (STRICT mode).
Expected: no ERROR findings. Acceptable: INFO/WARNING about absent terminology bindings (none are required for this minimal local-coded model). Common things the linter checks and that this file already satisfies: inner archetype id matches filename; every node has a `term_definitions` entry; no at-code is defined-but-unused or used-but-undefined; `occurrences` vs `cardinality` used correctly.

- [ ] **Step 5: Fix any ERROR findings and re-lint**

If the linter reports an ERROR (e.g. `lifecycle_state` value not accepted, or an at-code mismatch), fix it inline and re-run `/archetype-lint` until clean. Likely small fixes only: if `"unmanaged"` is rejected, change `lifecycle_state` to `<"draft">`; if a `C_DV_QUANTITY` form is flagged, compare byte-for-byte against the `at0094` block in `local/openEHR-EHR-OBSERVATION.ecg_result.v1.adl`.

- [ ] **Step 6: Commit**

```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
git add "local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl"
git commit -m "Add OBSERVATION.six_minute_walk_test.v0 archetype"
```

---

## Task 2: Author the template

**Files:**
- Create: `local/6 Minute Walk Test.t.json`
- Reference: `local/openEHR-EHR-COMPOSITION.encounter.v1.adl`, `local/Vital signs.t.json`

> **Note on template generation.** openEHR templates in this repo carry tool-computed `MD5-CAM`/`buildUid` checksums and a full operational definition tree. Per the repo CLAUDE.md these MUST NOT be hand-fabricated. There are two valid paths; pick **2A (tooling)** if an Archetype Designer is available, otherwise **2B (minimal hand-authored)**.

### Path 2A — Generate with Archetype Designer (preferred)

- [ ] **Step 1: Import archetypes**

In Archetype Designer (e.g. tools.openehr.org / Better Archetype Designer), import a working set containing `local/openEHR-EHR-COMPOSITION.encounter.v1.adl` and the new `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`.

- [ ] **Step 2: Build the template**

Create a new template named `6 Minute Walk Test`, root archetype `openEHR-EHR-COMPOSITION.encounter.v1`. Add `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0` as content. Apply no narrowing (minimal template). Set original language `en`.

- [ ] **Step 3: Export**

Export as the Web Template JSON used in this repo (top-level `"@type": "TEMPLATE"`) and save to `local/6 Minute Walk Test.t.json`. The tool fills `uid`, `buildUid`, `templateId`, `definition`, `terminology`, and `MD5-CAM` hashes correctly — do not edit them.

- [ ] **Step 4: Validate** — go to Task 2 "Shared validation" below.

### Path 2B — Minimal hand-authored web template (fallback, no GUI)

- [ ] **Step 1: Generate a fresh UID**

Run: `uuidgen`
Expected: a UUID; substitute for `<GENERATED-TEMPLATE-UID>` below.

- [ ] **Step 2: Write a minimal template file**

Create `local/6 Minute Walk Test.t.json` with this content. This is the lightweight "simplified" form (no fabricated `MD5-CAM`/`buildUid` — those are added when an OPT compiler first processes the template; omitting them is correct, fabricating them is not):

```json
{
  "@type": "TEMPLATE",
  "uid": "<GENERATED-TEMPLATE-UID>",
  "templateId": "6 Minute Walk Test",
  "description": {
    "@type": "RESOURCE_DESCRIPTION",
    "originalAuthor": {
      "date": "2026-06-08",
      "name": "Sebastian Iancu",
      "organisation": "Code24",
      "email": "sebastian.iancu@code24.nl"
    },
    "lifecycleState": {
      "codeString": "unmanaged"
    },
    "otherDetails": {
      "original_language": "ISO_639-1::en"
    },
    "details": {
      "en": {
        "@type": "RESOURCE_DESCRIPTION_ITEM",
        "language": {
          "terminologyId": { "value": "ISO_639-1" },
          "codeString": "en"
        },
        "purpose": "A minimal template recording a 6-minute walk test (distance walked, duration, laps, Borg breathlessness, walking aid, course length) within an encounter."
      }
    }
  },
  "originalLanguage": "ISO_639-1::en",
  "adlVersion": "1.4",
  "rmName": "openEHR",
  "definition": {
    "rmType": "COMPOSITION",
    "nodeId": "openEHR-EHR-COMPOSITION.encounter.v1",
    "name": "6 Minute Walk Test",
    "min": 1,
    "max": 1,
    "children": [
      {
        "rmType": "OBSERVATION",
        "nodeId": "openEHR-EHR-OBSERVATION.six_minute_walk_test.v0",
        "name": "6 Minute Walk Test",
        "min": 0,
        "max": 1
      }
    ]
  }
}
```

- [ ] **Step 3: Validate** — continue to "Shared validation" below.

### Shared validation (both paths)

- [ ] **Step 1: JSON is well-formed and top-level type is correct**

Run:
```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
jq -e '."@type" == "TEMPLATE"' "local/6 Minute Walk Test.t.json"
```
Expected: prints `true`, exit code 0.

- [ ] **Step 2: Referential integrity — template references only archetypes present in `local/`**

Run:
```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
grep -oE 'openEHR-EHR-(COMPOSITION|OBSERVATION)\.[a-z_]+\.v[0-9]+' "local/6 Minute Walk Test.t.json" | sort -u
ls local/openEHR-EHR-COMPOSITION.encounter.v1.adl local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl
```
Expected: the referenced ids are exactly `openEHR-EHR-COMPOSITION.encounter.v1` and `openEHR-EHR-OBSERVATION.six_minute_walk_test.v0`, and both `.adl` files exist.

- [ ] **Step 3: Explain the template to confirm it composes as intended**

Invoke the workspace command `/template-explain` on `local/6 Minute Walk Test.t.json`.
Expected: it reports a COMPOSITION (encounter) containing the 6 Minute Walk Test OBSERVATION with the distance/duration/laps/Borg/walking-aid/course-length nodes. Fix any reported broken reference before committing.

- [ ] **Step 4: Commit**

```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
git add "local/6 Minute Walk Test.t.json"
git commit -m "Add 6 Minute Walk Test template"
```

---

## Task 3: Final verification of the artifact set

**Files:** none (verification only)

- [ ] **Step 1: Re-lint the archetype clean**

Invoke `/archetype-lint` once more on `local/openEHR-EHR-OBSERVATION.six_minute_walk_test.v0.adl`.
Expected: no ERROR findings (matches Task 1, Step 4).

- [ ] **Step 2: Confirm both files are committed and tree is clean**

Run:
```bash
cd /src/sebastian-iancu/openehr-assistant-sandbox
git status --short
git log --oneline -3
```
Expected: `git status --short` is empty; the last two commits are the template and the archetype.

- [ ] **Step 3: Confirm the spec's validation criteria are met**

Check each item in spec §6 against the result: archetype lints clean; inner id matches filename; every node has a `term_definitions` entry; both coded nodes (Borg ordinal, walking aid) define their codes inline; the template references only archetypes present in `local/`. Note any deviation in the commit message or a follow-up.

---

## Self-Review (completed by plan author)

**1. Spec coverage:** distance (at0004), duration (at0005), laps (at0006), inline Borg ordinal (at0007/at0013–23), walking aid coded text (at0009/at0024–29), course length (at0011), comment (at0012), single point-event (at0002), template on encounter — all map to spec §3/§4. Out-of-scope items (vitals, O₂, synopsis, stop episodes) are intentionally absent per spec §5. ✓

**2. Placeholder scan:** `<GENERATED-UID>` and `<GENERATED-TEMPLATE-UID>` are generate-at-author-time values with an exact `uuidgen` command, not vague TBDs. No "add error handling"/"similar to"/"write tests for the above" placeholders. ✓

**3. Type/at-code consistency:** the at-code table, the `definition` block, and the `term_definitions` block all use at0000–at0029 with identical rubrics; the template's `nodeId`s match the archetype id and the encounter id exactly. Borg ordinal codes at0013–at0023 (11 codes = values 0–10) match the 11 `value matches` lines. ✓

**Deviation from spec, recorded:** spec §4.1 mentioned "value_sets" and an assumed value "none"; implemented as inline `[local:: ...]` defining-code lists (repo convention) with **no** `assumed_value` (kept minimal / lint-safe). Functionally equivalent for this minimal model.
