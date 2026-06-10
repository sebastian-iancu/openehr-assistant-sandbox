# openEHR Assistant — translation / localisation improvements

**Audience:** an AI refactor agent working on `openehr-assistant-mcp` (the MCP server) and `openehr-assistant-plugin` (the Claude Code plugin: skills, references, commands).
**Source:** the 2026-06-09 sandbox session that added a Dutch (`nl`) locale to `CLUSTER.exam.v2`, `OBSERVATION.exam.v1`, and `OBSERVATION.imaging_exam_result.v1`. See the companion retrospective: [`docs/blog/2026-06-09-dutch-translation-exam-archetypes-with-openehr-assistant.md`](../blog/2026-06-09-dutch-translation-exam-archetypes-with-openehr-assistant.md).
**Status:** proposals for review. Nothing here is a defect in the translated archetypes themselves (those passed the gate); these are toolchain gaps observed while doing the work.

## Ground rules

- Do **not** weaken the existing "translations edit text only" guarantees; every proposal must keep at-code parity and additions-only behaviour as invariants.
- Prefer additive tools/guides over breaking changes to existing tool signatures.
- Keep `(needs review)` honesty: no proposal may cause the toolchain to emit unverified Dutch (or any locale) terms as if they were authoritative.
- Faithful scope: each finding below was actually hit during the session, not hypothesised.

## Findings summary

| # | Pri | Repo | Theme | One-line |
|---|-----|------|-------|----------|
| F1 | P0 | mcp | Validation | No programmatic lint/parity check; the translation gate is manual |
| F2 | P1 | plugin (+mcp) | Scaffolding | No target-locale scaffold; at-code drift only caught after the fact |
| F3 | P1 | mcp (+plugin) | Terminology | `language-standards-nl` glossary too small; no NL display-term lookup |
| F4 | P1 | mcp | Reuse | `ckm_archetype_get` can't tell me if an upstream `nl` translation exists |
| F5 | P2 | mcp | Checksums | No recompute helper for `MD5-CAM` / `build_uid` after edits |
| F6 | P2 | mcp | Large files | No "return one ADL section as structured data" affordance |
| F7 | P2 | plugin | Docs | `translation.md` references the base guide but the `nl` guide is the one loaded |

---

## P0 — validation closes the loop

### F1. Add an `archetype_lint` (and translation-parity) MCP tool

- **Repo:** `openehr-assistant-mcp`
- **Problem:** `references/translation.md` mandates a "lint pass" whose failure "blocks done", but the only linter is the manual `archetype-lint` *skill*. In this session every gate check (at-code parity per language, `<`/`>` balance, name-vs-description punctuation) had to be re-implemented ad hoc in `awk`/`grep`. There is no callable tool that turns an `.adl` into structured findings.
- **Proposed change:** Expose `archetype_lint(path|content, mode=strict|permissive)` returning structured findings `{rule_id, severity, at_code?, language?, message, line?}`. Implement at minimum the rules already documented in the `archetype-lint` skill, plus a dedicated **translation-parity** check: for every language block in `ontology.term_definitions`, assert the at-code set equals the original-language set, and report adds/drops. Bonus: delimiter balance and the sentence-case / terminal-punctuation rules.
- **Acceptance criteria:**
  - Given today's three translated files → all three report `pass` with `nl`↔`en` parity (10/14/69).
  - Artificially deleting one `nl` at-code → tool flags `parity: nl missing atXXXX` at ERROR.
  - Appending a full stop to an `nl` `text` label → tool flags the naming-punctuation rule.
- **Effort / risk:** M / Low (read-only; no file mutation).

---

## P1 — make translating safe and grounded

### F2. Translation scaffolding (`/archetype-translate-scaffold` + MCP support)

- **Repo:** `openehr-assistant-plugin` (command/skill), backed by an `openehr-assistant-mcp` helper.
- **Problem:** The `nl` `term_definitions` block was hand-built in source order; the only thing preventing at-code drift was after-the-fact `awk`. Prevention should be up front.
- **Proposed change:** Given `(archetype, target_lang)`, emit a ready-to-paste skeleton: the `["xx"]` author stub for `language.translations`, the `details` stub, and a `term_definitions["xx"]` block pre-seeded with **every** source at-code in source order, `text`/`description`/`comment` placeholders preserved where the source has them, and the three insertion anchors identified by line. The translator fills text only; parity is structural, not vigilance-based.
- **Acceptance criteria:**
  - Scaffold for imaging emits exactly 69 at-code stubs in `en` order, including `comment` slots only where `en` has them.
  - Pasting the filled scaffold and running F1 → parity `pass` with no manual reordering.
- **Effort / risk:** M / Low.

### F3. Grow the Dutch glossary and wire NL terminology lookup

- **Repo:** `openehr-assistant-mcp` (guide content + `terminology_resolve`).
- **Problem:** `language-standards-nl` is explicitly a *seed*. The imaging archetype needed terms it doesn't contain — *Target body site*, *Device*, *Stabilising appliance*, *Study instance identifier* — all shipped `(needs review)`. `terminology_resolve` exists but did not offer a path to confirm Dutch **display terms** from SNOMED CT NL / Nictiz zib's.
- **Proposed change:** (a) Extend the `nl` glossary with the exam/imaging terms confirmed in this session (see backlog below), each tagged with its source; (b) teach `terminology_resolve` (or add `terminology_resolve(..., display_language="nl")`) to return the Dutch preferred term for a bound concept, so a translator can discharge `(needs review)` without leaving the toolchain. Keep the no-fabrication rule: unresolved → return "unconfirmed", never a guess.
- **Acceptance criteria:**
  - `language-standards-nl` glossary contains entries for the four flagged concepts with a `source` field.
  - `terminology_resolve` for a SNOMED CT concept with `display_language=nl` returns the NL preferred term or an explicit `unconfirmed`.
- **Effort / risk:** M / Medium (terminology sourcing must be authoritative, not invented).

### F4. Reuse-first for translations in `ckm_archetype_get`

- **Repo:** `openehr-assistant-mcp`
- **Problem:** The skill's reuse-first principle has no translation analogue. Nothing surfaced whether a published `nl` translation of these exact archetypes already exists upstream that should have been imported rather than re-translated.
- **Proposed change:** Have `ckm_archetype_get` expose the set of languages present in the fetched artefact's `term_definitions` (and ideally return a requested language's block directly), so the workflow can check "does an `nl` translation already exist upstream?" before translating from scratch.
- **Acceptance criteria:**
  - `ckm_archetype_get(id)` response includes `languages: [en, de, sv, …]` for the artefact.
  - A documented step in `archetype-authoring` (translate path) tells the agent to check this before authoring a new locale.
- **Effort / risk:** S / Low.

---

## P2 — quality-of-life

### F5. Checksum recompute helper

- **Repo:** `openehr-assistant-mcp`
- **Problem:** Editing staled `MD5-CAM-1.0.1` / `build_uid` / `revision`; the guide (correctly) forbids fabricating them, leaving the files with knowingly-stale metadata and no remedy in-toolchain.
- **Proposed change:** Add `archetype_recompute_checksum(content)` implementing the CKM CAM digest algorithm (or clearly document that recomputation is out of scope and must run in CKM/ADL tooling, and have F1 emit a `stale-checksum` INFO so it's at least surfaced).
- **Acceptance criteria:** Either the tool reproduces CKM's `MD5-CAM-1.0.1` for an unedited published file, or F1 reports `stale-checksum` after any text edit. Never emits a fabricated value silently.
- **Effort / risk:** M / Medium (must match CKM's canonical form exactly) — or S / Low for the surface-only variant.

### F6. Section-scoped ADL reader

- **Repo:** `openehr-assistant-mcp`
- **Problem:** The imaging file (2443 lines / ~63k tokens) exceeded the Read cap; mapping it meant paging + `grep`. Reconstructing structure from line offsets is error-prone.
- **Proposed change:** `archetype_section_get(path, section)` returning `definition` / `ontology` / `term_definitions[lang]` / `details[lang]` as structured data, so an agent can pull just the `en` `term_definitions` to translate without ingesting the whole file.
- **Acceptance criteria:** `term_definitions[en]` for imaging returns 69 items as `{at_code, text, description, comment?}` without the agent reading the full file.
- **Effort / risk:** M / Low.

### F7. Reference/guide loading nit

- **Repo:** `openehr-assistant-plugin`
- **Problem:** `references/translation.md` step 1 says load `archetypes/language-standards` *and* the per-language guide; in practice the `nl` guide (which states "all base principles A–F apply") is sufficient and is what got loaded. The base load was redundant for this task.
- **Proposed change:** Have `translation.md` note that the per-language guide subsumes the base sections it needs, or have each per-language guide explicitly enumerate which base sections are load-bearing — so agents don't double-load.
- **Effort / risk:** S / Low (doc only).

---

## Content-fix backlog (the artefacts touched this session)

These are about the three translated files, for a human/terminology reviewer — **not** blockers; the translations passed the structural gate.

1. **Confirm `(needs review)` Dutch terms** against Nictiz zib's / SNOMED CT NL and promote (or correct) in both the files and the `nl` glossary:
   - `imaging_exam_result` at0055 *Target body site* → "Beoogde lichaamslocatie"; at0006 → "Gestructureerde beoogde lichaamslocatie".
   - `imaging_exam_result` at0069 *Device* → "Apparaat" (decide vs "Hulpmiddel"; align with the Dutch label of `CLUSTER.device`).
   - `exam.v1` at0010 *Device Details* → "Apparaatdetails" (same Device decision).
   - `imaging_exam_result` at0107 *Stabilising appliance* → "Stabiliserend hulpmiddel".
   - `imaging_exam_result` at0092 *Study instance identifier* → "Onderzoeksinstantie-identificatie".
   - Metadata openers ("Vastleggen van… / Gebruik om… / Niet gebruiken om…") — confirm house form against published `nl` CKM metadata.
2. **Stale build metadata** on all three files (`MD5-CAM-1.0.1`, `build_uid`, `revision`) must be recomputed by CKM/ADL tooling before any publish (see F5).

## Implementation order

1. **F1** (validation) — unblocks everything; lets every later change be regression-tested.
2. **F2** (scaffold) — depends on F1's parity check for its acceptance test.
3. **F4** (reuse check) — small, independent, high principle-value.
4. **F3** (glossary + NL terminology) — discharges the content backlog item 1.
5. **F6**, **F5**, **F7** — quality-of-life, any order.

## Acceptance test pack

- **AT-1 (parity):** Run F1 on the three session files → `nl`↔`en` parity 10/14/69, all `pass`.
- **AT-2 (regression):** Delete one `nl` at-code → F1 ERROR; restore → `pass`.
- **AT-3 (scaffold):** F2 for imaging → 69 ordered stubs, `comment` slots only where `en` has them; fill + F1 → `pass`.
- **AT-4 (terminology honesty):** `terminology_resolve(display_language=nl)` on an unmapped concept → `unconfirmed`, never a fabricated term.
- **AT-5 (reuse):** `ckm_archetype_get` on a multi-language artefact → `languages` list present and correct.
- **AT-6 (additions-only):** Any tool-assisted translation flow → `git diff --numstat` shows 0 deletions on the target file.
