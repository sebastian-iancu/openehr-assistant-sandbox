# Refactor brief — openEHR Assistant translation workflow

**Audience:** an AI coding agent working in the `openehr-assistant-mcp` (PHP MCP
server) and `openehr-assistant-plugin` (Claude Code plugin: commands + skills) repos.
**Source:** observations from a real task in the `openehr-assistant-sandbox` —
adding a Dutch (`nl`) locale to three published exam/imaging archetypes
(`CLUSTER.exam.v2`, `OBSERVATION.exam.v1`, `OBSERVATION.imaging_exam_result.v1`).
**Date:** 2026-06-09.

> Read the companion retrospective at
> `docs/blog/2026-06-09-translating-exam-archetypes-to-dutch.md` for narrative context.

## How to use this brief

- Each task has an **ID**, a **target repo**, a **confidence** (HIGH = directly
  observed gap; MED = inferred; LOW = nice-to-have), the **problem**, the **change**,
  a **suggested location**, and **acceptance criteria**.
- **Use the `openehr-assistant-dev` plugin skills** to do the work, not ad-hoc edits:
  - `guide-prompt-authoring` → new/edited guides under `resources/guides/`.
  - `mcp-tool-authoring` → new `#[McpTool]`/`#[McpResource]` + PHPUnit tests.
  - `example-authoring` → if adding worked examples.
  - `release-workflow` → SemVer bump, Keep-a-Changelog entry, and the
    **mcp ↔ plugin compatibility sync** that several of these tasks require.
- Paths marked *(verify)* are my best inference from tool/skill metadata, not confirmed
  reads. Confirm before editing.
- **Do not** change the normative *content* rules of `language-standards.md`
  (sections A–H); these tasks extend tooling and add a locale guide, they do not
  rewrite policy.

---

## Findings (what the task exposed)

1. **No Dutch language-standards guide.** The base guide
   (`openehr://guides/archetypes/language-standards`) section K lists only Norwegian
   Bokmål (`nb`) as an existing per-language extension. Dutch term choices had to be
   improvised from general clinical register plus the `de`/`nb`/`sv` blocks already in
   the files. *(HIGH — confirmed by reading section K.)*
2. **The `/archetype-translate` skill has no verification step.** Its "Required Output"
   is (1) full ADL, (2) mapping table, (3) warnings — but nothing machine-checkable. The
   single most valuable safety net on the task (at-code parity between source and target
   locale, delimiter balance) was done by hand. *(HIGH — confirmed from skill text.)*
3. **No translation-reuse path.** Nothing in the workflow queries CKM (or a national
   release) for an existing locale block before minting new terms — counter to openEHR's
   reuse-first ethos applied to translations. *(MED — inferred.)*
4. **Metadata goes stale silently.** Adding a locale should regenerate
   `MD5-CAM-1.0.1` and bump `revision`; the repo convention forbids hand-fabricating
   hashes and no tool recomputes them. *(MED.)*
5. **Validation is ad-hoc and ADL is tab-sensitive.** `/archetype-lint` exists but the
   translate flow doesn't invoke it, and text-anchored edits into a whitespace-sensitive
   format are fragile. *(MED.)*

---

## Tasks — openehr-assistant-mcp

### T1 — Author `language-standards-nl` guide  *(HIGH)*
**Problem:** Finding 1.
**Change:** Create a Dutch per-language guide modelled on the existing
`language-standards-nb` guide: Dutch clinical register guidance, a maintained glossary
of common archetype terms, Purpose/Use/Misuse exemplars in Dutch, and references to
authoritative Dutch sources (Nictiz; the Dutch SNOMED CT edition / Nationale Release).
Seed the glossary from the terms established on the task:

| English | Dutch | Note |
|---|---|---|
| Physical examination findings | Bevindingen lichamelijk onderzoek | |
| (System or) structure examined | Onderzocht systeem of structuur | |
| Examination detail / findings | Onderzoeksdetails / Onderzoeksbevindingen | |
| Body site / Structured body site | Lichaamslocatie / Gestructureerde lichaamslocatie | |
| Clinical interpretation | Klinische interpretatie | |
| Confounding factors | Verstorende factoren | |
| Comment | Commentaar | |
| Extension | Extensie | |
| Position (body position) | Lichaamshouding | label, not *Positie* |
| Imaging examination result | Resultaat beeldvormend onderzoek | |
| Modality | Modaliteit | |
| Study (DICOM) | onderzoek (label keeps *Study* in *Study Instance UID*) | flag for review |
| subject of care | zorgvrager | |
| Event Series / Tree / `@ internal @` | *(leave verbatim)* | internal RM nodes |

**Suggested location:** `resources/guides/archetypes/language-standards-nl.md` *(verify)*.
**Also:** add `nl` to section K ("Available Language Standards Guides") of
`resources/guides/archetypes/language-standards.md`.
**Acceptance:** `guide_get(category="archetypes", name="language-standards-nl")` resolves;
glossary present; base guide section K updated; use `guide-prompt-authoring`.

### T2 — Translation-reuse lookup tool  *(MED)*
**Problem:** Finding 3.
**Change:** Provide a way to fetch existing locale blocks for an archetype id so a
translator can inherit community terms. Either extend `ckm_archetype_get` with a
`locales`/`language` projection, or add a tool, e.g.
`ckm_archetype_translations(archetype_id, language?)` returning per-`at-code`
`text`/`description`/`comment` for the requested (or all) languages.
**Suggested location:** `src/Tools/` + PHPUnit under the mirrored test path *(verify)*.
**Acceptance:** given an archetype id with multiple locales, the tool returns the target
language block (or reports its absence); covered by tests; use `mcp-tool-authoring`.
**Verify first:** whether `ckm_archetype_get` already returns full multi-locale ADL
(if so, this may only need documentation + a skill step, not a new tool).

### T3 — Archetype metadata maintenance tool  *(MED)*
**Problem:** Finding 4.
**Change:** Add a tool to recompute `MD5-CAM-1.0.1` and bump `revision` (and optionally
`build_uid`) after an edit, e.g. `archetype_metadata_refresh(adl)`.
**Risk / verify:** `MD5-CAM-1.0.1` is CKM's canonical checksum algorithm — **do not
guess it**. Source the exact algorithm/normalisation from the openEHR CKM / AOM
reference before implementing; if it cannot be confirmed, ship only the `revision` bump
and emit a warning that the hash needs regeneration at CKM import.
**Suggested location:** `src/Tools/` + tests *(verify)*.
**Acceptance:** for a known archetype, recomputed hash matches CKM's published value
(round-trip test) **or** the tool clearly degrades to revision-only with a warning.

### T4 — Surface target-language preferred terms  *(LOW)*
**Problem:** Finding 5 (terminological grounding).
**Change:** If `terminology_resolve` supports language designations, expose target-language
preferred terms so translators can anchor labels (esp. SNOMED CT) instead of relying on
judgement. Otherwise document the limitation.
**Acceptance:** a resolve call can return e.g. a Dutch preferred term for a bound concept,
or the gap is documented.

---

## Tasks — openehr-assistant-plugin

### T5 — Add a verification step to `/archetype-translate`  *(HIGH)*
**Problem:** Finding 2.
**Change:** Extend the `/archetype-translate` command/skill with a mandatory
post-translation verification, added to "Required Output":
- **at-code parity** — the set of `at####` in the target `term_definitions` block MUST
  equal the source language's set (no missing, extra, duplicated, or invented codes).
- **delimiter balance** — `<` count equals `>` count in the file.
- **untouched invariants** — assert no diff to at-codes, occurrences/cardinalities,
  units, value sets, or `term_bindings` (text-only change).
- **auto-lint** — invoke `/archetype-lint` on the patched file and surface results.
**Suggested location:** the `/archetype-translate` command definition (likely
`commands/archetype-translate.md`) *(verify)*.
**Acceptance:** running the skill produces a verification block with the parity count
(e.g. `en vs nl: 69 = 69`) and a lint summary; failures block the "done" claim.

### T6 — Document the edit mechanics for tab-sensitive ADL  *(MED)*
**Problem:** Finding 5.
**Change:** In the skill, prescribe the **three insertion locations**
(`language.translations`, `description.details`, `ontology.term_definitions`) and the
**"insert at the top of the block"** strategy (anchor on the block opener + first child,
prepend the new locale) to avoid brittle matching of nested closing-delimiter runs.
Note tab (not space) indentation and the per-level indentation depths.
**Acceptance:** skill text documents all three locations and the insertion strategy.

### T7 — Add reuse-first + metadata steps to the skill  *(MED)*
**Problem:** Findings 3 and 4.
**Change:** Before translating, instruct the skill to check for an existing target-locale
block via T2 (reuse community terms where present). After translating, instruct it to
flag `MD5-CAM-1.0.1`/`revision` as stale and call T3 when available.
**Acceptance:** skill references the reuse lookup and the metadata refresh; emits a stale-
metadata warning when the tool is unavailable.

---

## Cross-repo / release

### T8 — Compatibility + changelog  *(required if any tool/skill changes ship)*
Use `release-workflow`: bump SemVer, add Keep-a-Changelog entries in both repos, and keep
the **mcp ↔ plugin compatibility pointer** and any `allowed-tools` / manifests in sync
(new MCP tools from T2/T3 must be reflected in the plugin's tool allowlist).

---

## Suggested order

1. **T1** (highest value, lowest risk; pure content via `guide-prompt-authoring`).
2. **T5** (cheap, high-safety skill change).
3. **T6 / T7** (skill polish).
4. **T2** (verify `ckm_archetype_get` first — may shrink to docs).
5. **T3** (gated on confirming the MD5-CAM algorithm).
6. **T4** (opportunistic).
7. **T8** whenever tooling changes are bundled for release.

## Open questions to resolve before coding

- Does `ckm_archetype_get` already return full multi-locale ADL? (gates T2)
- What is the exact `MD5-CAM-1.0.1` normalisation/algorithm? (gates T3)
- Does `terminology_resolve` accept a target-language parameter? (gates T4)
- Confirm real paths for the plugin command file and the MCP `src/Tools/` + guide dirs.
