# Reviewing a published CKM archetype with the openEHR Assistant — a session retrospective

*2026-06-09 · a working log of how an AI pair-modeller took "check the quality of `CLUSTER.anatomical_location.v1`" through lint, rationale, explanation, CKM survey, and workspace impact analysis — and what the toolchain did well versus where it still asks the model to improvise.*

---

## The request

The session began with a focused quality question:

> *"Use your skills and check review and assess quality of `local/openEHR-EHR-CLUSTER.anatomical_location.v1.adl`."*

That sounds like a single pass over one file. In practice it became a **full archetype lifecycle walk-through** on a mature, published artefact — the kind of CLUSTER that sits inside dozens of CKM archetypes and templates worldwide. The user then asked for rationale prose, a semantic explanation, similar CKM artefacts, and a local impact analysis. Together, those map cleanly onto the plugin's review commands: `/archetype-lint`, `/archetype-rationale`, `/archetype-explain`, and `/archetype-impact`.

The archetype under review is not a sandbox draft. It is the openEHR Foundation's canonical **Anatomical location** CLUSTER (CKM revision **1.5.0**, `lifecycle_state = published`), with the same UID as the curated gold-standard example at `openehr://examples/archetypes/openEHR-EHR-CLUSTER.anatomical_location.v1`.

## How it was solved

Work followed the plugin's documented order of operations — guides first, tools second, interpretation last.

### 1. Quality review and lint (`/archetype-lint` skill)

The `archetype-lint` skill drove the first deliverable:

1. Load normative guides via MCP: `archetypes/rules`, `anti-patterns`, `checklist`.
2. Read the ADL in sections (the file is **2 142 lines** — too large for a single read).
3. Apply the 22 lint rules manually against structure, terminology, slots, and translations.
4. Cross-check CKM via `ckm_archetype_search` and the examples corpus via `examples_search`.

**Verdict (PERMISSIVE mode): PASS** — 0 ERRORs, 5 WARNINGs, 1 INFO. Structurally sound; editorial/i18n debt typical of a long-lived CKM artefact (Catalan `at0067` mistranslation, incomplete translations, `clock` vs `circle` prose mismatch).

### 2. Rationale and clinical meaning

The user asked *why* the archetype exists. That was answered from the archetype's own `purpose`, `use`, and `misuse` text plus the postcoordination pattern in `definition` (Body site name + optional laterality, aspect, specific site, anatomical line, description, slots).

A later `/archetype-rationale` command produced CKM-quality prose for `description`, `purpose`, `misuse`, and `use`, aligned with `language-standards` and sibling style from `CLUSTER.anatomical_location_relative.v2`.

### 3. Semantic explanation (`/archetype-explain`)

A structured explanation covered: high-level clinical meaning, mandatory vs optional data elements, terminology bindings (SNOMED for laterality only), slot constraints, and semantic boundaries deferred to templates.

### 4. CKM reuse survey

`ckm_archetype_search` surfaced the **anatomical location family**:

| Archetype | Status | Role |
|---|---|---|
| `CLUSTER.anatomical_location.v1` | Published | Base — named site + qualifiers |
| `CLUSTER.anatomical_location_relative.v2` | Published | Landmark-relative precision |
| `CLUSTER.anatomical_location_circle.v1` | Published | Circular/coordinate precision |
| `CLUSTER.anatomical_location_precise.v0` | Draft | Superseded experiment |
| Dentistry-specific `*_tooth` / `*_gingivae` | Initial | Domain specialisations |

No second published general-purpose macroscopic location CLUSTER duplicates the base archetype.

### 5. Workspace impact (`/archetype-impact`)

A workspace grep found:

- **1 template:** `Health Certificate.t.json` — overlay on `anatomical_location.v1`, nested under `CLUSTER.exam.v2`.
- **3 parent archetypes** with slot constraints: `exam.v2`, `problem_diagnosis.v1`, `imaging_exam_result.v1`.
- **0** AQL/SQL/md references.

Impact in this sandbox is small; in CKM production the blast radius would be enormous.

## What the openEHR Assistant MCP, plugin, and skills contributed

### MCP server (`openehr-assistant-mcp`)

| Tool / resource | How it was used |
|---|---|
| `guide_get` | Loaded `archetypes/rules`, `principles`, `structural-constraints`, `terminology`, `anti-patterns`, `checklist`, `language-standards` — the normative backbone for lint, rationale, and explain outputs. |
| `ckm_archetype_search` | Discovered the location family and confirmed published status/revisions. |
| `ckm_archetype_get` | Fetched full ADL for CKM identity and sibling `anatomical_location_relative.v2` prose style. |
| `examples_search` | Confirmed the local file matches the curated gold-standard example (same UID). |
| `type_specification_get` | Clarified `CLUSTER` RM semantics (`items` cardinality `1..*`). |

Guides are the standout asset: they turn "review this file" into a **checklist-backed audit** rather than opinion. CKM search grounded reuse and family relationships with live metadata (status, revision, modification time).

### Plugin skills and commands

| Skill / command | Role in session |
|---|---|
| `archetype-lint` | Defined the 22-rule report format, PERMISSIVE vs STRICT gating, and mandatory guide loading. |
| `archetype-authoring` | Pointed to gold-standard examples and CKM reuse-first workflow. |
| `/archetype-rationale` | Structured CKM editorial prose with British English and sibling style matching. |
| `/archetype-explain` | Fixed output sections — no improvement suggestions, template behaviour out of scope. |
| `/archetype-impact` | Drove workspace artefact discovery (templates, AQL, docs). |

The plugin carried **domain methodology**; the MCP carried **retrieved facts**. That split worked when both were available.

## Shortcomings — what was missing or awkward

Honest friction log:

### 1. No machine-backed archetype lint tool

`/archetype-lint` is an LLM applying 22 normative **modelling** rules from guides. It is not a registered MCP tool and not an ADL parser. The session could not call `archetype_lint(path)` and get deterministic violations. Rule application depended on careful reading of a 2 000-line file. A wrapped lint engine (or Archie parse + rule engine) would make review reproducible.

### 2. MCP parameter mismatches — repeated trial and error

Several tool calls failed on the first attempt because parameter names in skill examples do not always match tool schemas:

| Tool | Skill / instinct said | Schema requires |
|---|---|---|
| `ckm_archetype_search` | `query` | `keyword` |
| `ckm_archetype_get` | `cid` | `identifier` |
| `type_specification_get` | `type_name` | `name` |

Each failure cost a round trip. Better: tool descriptors surfaced before first call, or skills generated from the live schema.

### 3. `terminology_resolve` does not cover SNOMED CT

The `/archetype-rationale` command asks to ground prose in at least two bound terminology codes. `anatomical_location.v1` binds laterality to SNOMED CT (`272741003`, `7771000`, `24028007`, `51440002`). `terminology_resolve` only handles **openEHR terminology** — SNOMED inputs returned "Could not resolve." Prose was grounded in archetype term definitions instead; SNOMED semantic verification was skipped.

### 4. Large archetype handling is manual

At 152 KB / 2 142 lines, the file exceeded the read tool's single-shot limit. The agent chunked reads and ran a Python script to verify at-code completeness. There is no MCP tool for "structure digest" (node tree, slot list, binding summary, translation completeness matrix) — useful for any published CKM archetype.

### 5. Sibling archetype fetch blocked by auto-review

Fetching `CLUSTER.device.v1` as a prose-style sibling was rejected as scope-widening. `anatomical_location_relative.v2` worked as a substitute. For `/archetype-rationale`, the command should name acceptable siblings explicitly or allow a read-only CKM fetch without classifier blocking.

### 6. `/archetype-impact` format vs workspace reality

The impact command globs `*.oet` and `*.opt`. This sandbox uses **`.t.json` web templates**. The grep still found `Health Certificate.t.json`, but only because a broader search was run. The command spec should include `*.t.json` (and optionally `*.adl` slot-constraint consumers).

### 7. No workspace-vs-CKM diff

A `diff` against CKM's latest ADL failed in the sandbox (network/path). Could not confirm whether `local/` is byte-identical to CKM revision 1.5.0 without a dedicated `ckm_archetype_diff` tool.

### 8. CKM-wide impact out of scope

`/archetype-impact` is honestly local-only. For `anatomical_location.v1`, the real dependency graph spans hundreds of CKM archetypes and templates. The command should either integrate a CKM reverse-dependency search or state prominently that local results are a lower bound.

### 9. Review vs edit boundary respected, but no lint autofix path

The lint skill correctly reports violations without modifying files. For the Catalan `at0067` error and `clock`/`circle` prose mismatch, fixes were described but not applied — appropriate for a review task, but a `lint --fix` or "minimal diff plan" MCP tool would shorten the loop when the user wants remediation.

## What I'd reach for next time

- **`archetype_lint` MCP tool** — deterministic 22-rule pass with file path in, violations JSON out.
- **`snomed_resolve` / extended `terminology_resolve`** — bind rationale and review to external code systems named in `term_bindings`.
- **`archetype_digest` MCP tool** — structure summary, at-code inventory, translation completeness, slot list for large ADLs.
- **`ckm_archetype_diff`** — local file vs CKM published revision.
- **`archetype_impact` MCP tool** — workspace glob including `.t.json` + optional CKM reverse-deps.
- **Schema-aligned skill examples** — generate skill allowed-tool docs from MCP tool JSON descriptors to stop parameter name drift.

## Takeaways

- **Published archetypes deserve review, not just authoring.** The toolchain is strong on *creating* artefacts; this session shows the lint/explain/impact commands are equally valuable on CKM imports already in `local/`.
- **Guides-first review scales.** Even without a parser, loading `rules` + `checklist` produced a defensible PASS/FAIL with cited rule IDs — closer to editorial QA than generic "looks fine."
- **Family context matters.** Understanding `anatomical_location` alongside `relative` and `circle` clarifies why slots are constrained the way they are — CKM search made that immediate.
- **Impact analysis exposes hidden coupling.** One template overlay and three slot-bearing parents in a small sandbox; multiply that by CKM's full graph for production change management.
- **The gaps are retrieval and determinism** — SNOMED grounding, large-file digests, machine lint, and CKM-wide dependency search — not clinical modelling knowledge. The domain content in guides and examples is already excellent.

---

*Primary artefact reviewed:* `local/openEHR-EHR-CLUSTER.anatomical_location.v1.adl`  
*Companion refactor brief:* `docs/improvements/2026-06-09-openehr-assistant-review-and-impact-improvements.md`
