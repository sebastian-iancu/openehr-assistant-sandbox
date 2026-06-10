# Translating the exam-family archetypes into Dutch with openEHR Assistant

*Three published openEHR archetypes, one new `nl` locale each, zero touched invariants — and what the toolchain did and didn't give me along the way.*

## The request

Translate three archetypes in `local/` into Dutch (`nl`):

- `openEHR-EHR-CLUSTER.exam.v2` — physical examination findings (10 at-codes)
- `openEHR-EHR-OBSERVATION.exam.v1` — physical examination findings, ENTRY-level (14 at-codes)
- `openEHR-EHR-OBSERVATION.imaging_exam_result.v1` — imaging examination result (69 at-codes)

All three are published, multi-language CKM artefacts (already carrying `de`, `sv`, `nb`, `en`, plus various of `ca`/`es`/`pt-br`/`el`/`sl`/`ar-sy`/`zh-cn`/`es-ar`). The job was purely additive: introduce a Dutch locale without disturbing the clinical model.

## How it was solved

1. **Routed to the right skill.** The request matched `archetype-authoring`, whose scope explicitly includes *translate / localise*. Invoking it first (before any file edit) was the non-negotiable move; it set the conflict-resolution priority (rules/structure > syntax > anti-patterns > principles > convenience) and pointed at the translation reference.
2. **Loaded the translation playbook.** `references/translation.md` defined the whole shape of the task: translations live in the **ontology block** (ADL 1.4, not a top-level `terminology` section); a locale must be inserted at the **top of three tab-sensitive blocks** — `language.translations`, `description.details`, `ontology.term_definitions`; and it specified a **machine-checkable "done" gate** (at-code parity, delimiter balance, untouched invariants, lint).
3. **Read and mapped the sources.** CLUSTER.exam was small enough to read whole. OBSERVATION.exam (1043 lines) and especially imaging (2443 lines / ~63k tokens) exceeded the Read cap, so I paged through and used `grep` to map every language block and the full at-code roster per file before writing a word of Dutch.
4. **Loaded the Dutch language standard.** `guide_get(archetypes/language-standards-nl)` was the keystone. Its **seed glossary** already mapped the exact concepts these archetypes use, and its rules covered sentence-case naming, compound formation (tussen-letters), de/het gender and adjective inflection, the metadata openers (*Vastleggen van… / Gebruik om… vast te leggen / Niet gebruiken om… vast te leggen*), prohibited practices, and a closing checklist.
5. **Translated, glossary-first.** For each file I added the `nl` author block, a translated `purpose`/`keywords`/`use`/`misuse`, and a full `term_definitions` set in the **same at-code order** as the English source — reusing glossary terms so shared concepts stayed identical across all three files. RM/structural labels (`Event Series`, `Tree`, `@ internal @`), class names (OBSERVATION/CLUSTER/ACTION…), and DICOM/FHIR attribute names (`Study Instance UID`, `DiagnosticReport.code`) were left verbatim per the guide.
6. **Verified against the gate.** `awk` confirmed at-code parity per language (10=10, 14=14, 69=69); `git diff --numstat` showed **additions only, 0 deletions** — proof the definition section, cardinalities, occurrences, value sets and `term_bindings` were untouched; a `<`/`>` count confirmed delimiter balance; and grep spot-checks confirmed no name label ends in a full stop while every purpose/use/misuse statement does.

## What the MCP / plugin / skills contributed

| Source | What it gave | Verdict |
|--------|--------------|---------|
| `archetype-authoring` skill | Routing, conflict-resolution priority, pointer to the translation reference | Essential entry point |
| `references/translation.md` | 3 insertion anchors, the verification gate, the stale-checksum advisory | The spec for the whole task |
| `guide_get(archetypes/language-standards-nl)` | Seed glossary, case/compound/gender rules, metadata openers, prohibited list, checklist | Single highest-value artefact |
| SessionStart hook | Detected the openEHR workspace (22 `.adl`), surfaced skills + commands up front | Useful orientation |

The glossary is what made the output *consistent* rather than merely *plausible*. Without it I would have had to coin "Bevindingen lichamelijk onderzoek", "Lichaamshouding" (not "Positie"), "Klinische interpretatie", "Verstorende factoren", "Resultaat beeldvormend onderzoek" independently in three files and hope they matched.

## Shortcomings

Honest gaps that cost time or left residual risk:

1. **The verification gate is documented but not automatable from inside the toolset.** `references/translation.md` says "run the lint pass… failures block done", but the only linter is the *manual* `archetype-lint` skill. I hand-rolled `awk`/`grep` equivalents for at-code parity and punctuation rules. There is no MCP tool that takes an `.adl` and returns structured lint findings.
2. **No translation scaffolding.** I built each `nl` `term_definitions` block by hand, in source order, with the risk of dropping or duplicating an at-code — a risk caught only by my own `awk` after the fact, not prevented up front.
3. **The seed glossary doesn't cover the imaging vocabulary.** *Target body site*, *Device*, *Stabilising appliance*, *Study instance identifier* aren't in it, so I flagged my renderings `(needs review)`. There was no grounded path to confirm them: `terminology_resolve` exists but isn't wired to surface SNOMED CT NL / Nictiz zib **Dutch display terms**, which is exactly what a translator needs.
4. **Reuse-first doesn't extend to translations.** The skill preaches reuse-first, and `ckm_archetype_get` returns native ADL — but nothing told me whether a published Dutch translation of these exact archetypes already exists upstream that I should have reused instead of translating fresh.
5. **Stale checksums, no recompute path.** Editing the files staled `MD5-CAM-1.0.1` / `build_uid` / `revision`. The guide is rightly emphatic not to fabricate them — but there's no helper to recompute them either, so the files now ship with knowingly-stale metadata pending upstream tooling.
6. **Large-file ergonomics.** The imaging file is 2443 lines; reading it meant paging plus `grep` reconnaissance. A tool returning *just* a named ADL section (e.g. the `en` `term_definitions` as structured data) would have been cheaper and less error-prone than reconstructing structure from line offsets.

## What I'd reach for next time

- `terminology_resolve` against SNOMED CT NL to discharge the `(needs review)` flags before calling the Dutch text canonical.
- An `examples_search(kind="archetypes")` pull of a known-good translated archetype as a structural model — skipped here only because the task was text-only and the glossary covered most of the concept set.

## Takeaways

- **Translation in ADL 1.4 is an ontology edit, not a terminology edit** — three blocks, tab-indented, source-order at-codes, text only.
- **A seed glossary beats a clever translator** for cross-file consistency; the most valuable thing the toolchain offered was a small table of confirmed renderings.
- **The "done" gate held**: at-code parity + additions-only diff + delimiter balance is a cheap, strong proof that a translation changed nothing it shouldn't have. The fact that I had to implement it by hand is the main thing the toolchain should fix.

---

*Primary artefacts:* [`local/openEHR-EHR-CLUSTER.exam.v2.adl`](../../local/openEHR-EHR-CLUSTER.exam.v2.adl), [`local/openEHR-EHR-OBSERVATION.exam.v1.adl`](../../local/openEHR-EHR-OBSERVATION.exam.v1.adl), [`local/openEHR-EHR-OBSERVATION.imaging_exam_result.v1.adl`](../../local/openEHR-EHR-OBSERVATION.imaging_exam_result.v1.adl)
*Companion brief:* [`docs/improvements/2026-06-09-openehr-assistant-translation-localisation-improvements.md`](../improvements/2026-06-09-openehr-assistant-translation-localisation-improvements.md)
