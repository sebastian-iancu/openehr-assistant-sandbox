---
title: "Adding a Dutch translation to three openEHR exam archetypes"
date: 2026-06-09
author: Sebastian Iancu
tags: [openehr, archetypes, adl, translation, dutch, claude-code, openehr-assistant-mcp]
---

# Adding a Dutch translation to three openEHR exam archetypes

A short field report on a translation task in this content sandbox: what was asked,
how it was done with the `openehr-assistant` plugin and MCP server, and — just as
usefully — where the toolchain came up short and what I wish had been on hand.

## The request

> "Translate to Dutch these two archetypes — `openEHR-EHR-CLUSTER.exam.v2.adl` and
> `openEHR-EHR-OBSERVATION.exam.v1.adl` — and also `openEHR-EHR-OBSERVATION.imaging_exam_result.v1.adl`."

Three published, multilingual ADL 1.4 archetypes, each already carrying several
language blocks (English original plus `de`, `sv`, `nb`, `pt-br`, `es`, `ca`, and
others). The job was to add a clinically faithful **Dutch (`nl`)** locale without
disturbing anything else.

These are not trivial files. Translatable text in an ADL 1.4 archetype lives in
**three** distinct places, and all three needed an `nl` sibling:

1. `language.translations` — the translator/locale metadata block.
2. `description.details` — `purpose`, `keywords`, `use`, `misuse`, `copyright`.
3. `ontology.term_definitions` — the `text` + `description` (+ `comment`) for every
   `at-code`.

By volume the third dominates: the imaging archetype alone defines **69 at-codes**,
including the FHIR/DICOM-aligned status value sets (`Registered`, `Partial`,
`Preliminary`, `Final`, `Amended`, `Corrected`, `Appended`, `Cancelled`, `Unknown`).

## How it was solved

### Skill-first, then guide-first

The work began by invoking the purpose-built **`/archetype-translate`** skill rather
than free-handing edits. That skill encodes the correct order of operations and,
critically, the *invariants* of a safe translation. It immediately routed me to the
authoritative reference via the MCP server:

```
guide_get(category="archetypes", name="language-standards")
```

That guide (sections A–L) is the single most valuable artefact I pulled this session.
The rules that actually shaped the output:

- **E1–E3** — translations are *not* literal; preserve clinical intent and use the
  target language's natural clinical register.
- **E5 / H4** — never alter at-codes, RM structure/paths, occurrences/cardinalities,
  units, value sets, or existing terminology bindings. Translation touches
  human-readable text *only*.
- **H3** — do not translate RM class names (`OBSERVATION`, `CLUSTER`, `ACTION`,
  `INSTRUCTION`) or archetype identifiers embedded in `use`/`misuse` prose.
- **D4 / F2** — spelling and terminology variants are a job for external bindings
  (SNOMED CT, LOINC), not for inventing new at-codes.

Section **K** also told me something by omission: the only per-language extension
guide that exists today is **Norwegian Bokmål** (`language-standards-nb`). There is
**no `language-standards-nl`** — a gap I'll return to.

### Translation mechanics

I translated from the English (`en`) source block in each file (the canonical
original per rule A1), not from one of the existing secondary locales, to avoid
translation-chain drift (rule A2). To keep the three files internally consistent I
fixed a small Dutch clinical glossary up front and applied it everywhere:

| English | Dutch |
|---|---|
| Physical examination findings | Bevindingen lichamelijk onderzoek |
| Examination detail / findings | Onderzoeksdetails / Onderzoeksbevindingen |
| Body site / Structured body site | Lichaamslocatie / Gestructureerde lichaamslocatie |
| Clinical interpretation | Klinische interpretatie |
| Confounding factors | Verstorende factoren |
| Imaging examination result | Resultaat beeldvormend onderzoek |
| Modality | Modaliteit |
| subject of care | zorgvrager |

Internal nodes (`Event Series`, `Tree`, `@ internal @`) were left verbatim, matching
every other locale in the files. DICOM/FHIR anchors (`DiagnosticReport.code`,
`ImagingStudy.modality`, `(0008,0090)`) were kept untouched inside the translated
comments — they are identifiers, not prose.

Edits were applied with an **"insert at the top of the block"** strategy: anchor on
the opening `... = <` of each block plus the first existing child, and prepend the new
`nl` block. This keeps the anchor short and unique and sidesteps the brittle business
of matching long, deeply-nested closing-delimiter runs in a tab-sensitive format.

### Verification

ADL 1.4 has no forgiving parser in the loop here, so I verified structurally before
claiming success:

| File | `<` / `>` balance | `nl` blocks | at-code parity (en vs nl) |
|---|---|---|---|
| `OBSERVATION.exam.v1` | 740 = 740 | 3 | 14 = 14 |
| `CLUSTER.exam.v2` | 445 = 445 | 3 | 10 = 10 |
| `OBSERVATION.imaging_exam_result.v1` | 1816 = 1816 | 3 | 69 = 69 |

The at-code parity check (a small Python pass comparing the `at####` set in the `en`
and `nl` `term_definitions` blocks) is the one that matters most: it proves no node
was dropped, duplicated, or invented during a 69-element translation.

### One follow-up

After review the reviewer asked to render the `Position` element label as
**`Lichaamshouding`** rather than `Positie`. Because both `OBSERVATION.exam.v1`
(at0013) and `imaging_exam_result.v1` (at0108) translate the same English term, the
change was applied to both for consistency.

## What the MCP server and plugin actually contributed

Being precise about this matters:

- **The `/archetype-translate` skill** supplied the *method* and the *guardrails* — the
  three-location structure, the don't-touch list, and the required outputs (full ADL,
  a translation mapping, and flagged warnings).
- **`guide_get` → `language-standards`** supplied the *normative rules* I translated
  against. This was the only MCP tool call that fed real content into the work.

Everything else — reading the source ADL, performing the edits, and the verification
pass — was local file and shell tooling. That's an honest picture: for a translation
task, the plugin's value was concentrated in *process discipline and a normative
reference*, not in bulk content generation.

## Shortcomings and what I missed

A candid list, because the gaps are the interesting part:

1. **No Dutch language-standards guide.** There is a rich `language-standards-nb`
   (Norwegian) with an established term glossary and maintenance procedure, but nothing
   for `nl`. I had to lean on general Dutch clinical register plus the `de`/`nb`/`sv`
   translations already embedded in the files. A `language-standards-nl` with agreed
   equivalents (ideally aligned with Nictiz / the Dutch SNOMED CT edition) would have
   removed real ambiguity — e.g. *Positie* vs *Lichaamshouding*, *onderzoek* vs
   *studie* for DICOM "study", and *zorgvrager* vs *subject van zorg*.

2. **No reuse step for existing Dutch translations.** openEHR's reuse-first ethos
   applies to *translations* too. I did not query CKM (or any Dutch national release)
   for prior `nl` renderings of these exact archetypes. If a published Dutch term set
   exists, I risked diverging from it. A "fetch existing translations for this archetype
   id" capability — analogous to `ckm_archetype_get` but locale-aware — would let a
   translator inherit community-blessed terms instead of minting new ones.

3. **No terminology grounding for labels.** I never called `terminology_resolve`.
   These exam archetypes bind little in their `term_definitions`, so it wasn't strictly
   required, but a lookup of Dutch SNOMED CT preferred terms would have given the
   clinical labels an external anchor rather than a translator's judgement.

4. **Checksum and revision are manual and were deliberately left alone.** CKM
   regenerates `MD5-CAM-1.0.1` and bumps `revision` when a locale is added. The repo
   convention says *don't hand-fabricate* hashes, and the toolchain offers no command to
   recompute them — so the metadata is now slightly behind the content until an import
   step fixes it. A "recompute MD5-CAM / bump revision" tool is an obvious missing piece.

5. **Validation was ad-hoc.** Bracket-balance and at-code-parity via grep/Python are a
   decent smoke test, but they are not a parse. `/archetype-lint` exists and I offered
   it but did not run it; a translation workflow that *automatically* re-lints the patched
   file would close the loop. Likewise, the tab-sensitivity of ADL 1.4 makes every Edit a
   small whitespace gamble that a structural editor would eliminate.

## Wishlist

If I could shape the next iteration of the toolchain for translation work specifically:

- A **`language-standards-nl`** guide with a maintained glossary (the `nb` guide is the
  template to follow).
- A **translation-reuse lookup** that returns any existing locale block for a given
  archetype id from CKM / national releases.
- A **metadata fixer** that recomputes `MD5-CAM-1.0.1` and bumps `revision` after a
  translation.
- An **ADL-aware patch + lint** path so insertions are structural (not text-anchored)
  and validation runs automatically.
- Optional **`terminology_resolve` integration** surfacing target-language preferred
  terms while translating labels.

## Takeaways

- The MCP plugin earned its keep here as a **discipline layer**: skill → guide → invariants
  → verification, in that order. The `language-standards` guide alone prevented the classic
  translation mistakes (touching at-codes, translating class names, inventing nodes).
- For a 69-element block, **machine-checkable invariants** (at-code parity, delimiter
  balance) beat eyeballing. They are cheap and they caught nothing — which is exactly the
  point of running them.
- The remaining risk is **terminological, not structural**: without a Dutch standards
  guide or a reuse path, term choices rest on the translator's register. That is where the
  toolchain has the most room to grow.

*Artefacts touched:* `local/openEHR-EHR-CLUSTER.exam.v2.adl`,
`local/openEHR-EHR-OBSERVATION.exam.v1.adl`,
`local/openEHR-EHR-OBSERVATION.imaging_exam_result.v1.adl`.
