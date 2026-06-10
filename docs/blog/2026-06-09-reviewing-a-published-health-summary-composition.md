# Reviewing a published COMPOSITION archetype — and discovering it was canonical all along

**Date:** 2026-06-09
**Status:** Session retrospective
**Author:** Sebastian Iancu (with Claude Code)
**Subject:** Design review + QA of [openEHR-EHR-COMPOSITION.health_summary.v1.adl](../../local/openEHR-EHR-COMPOSITION.health_summary.v1.adl)

## 1. The request

> *"Perform a design review and a QA check on `local/openEHR-EHR-COMPOSITION.health_summary.v1.adl`, spot issues and suggest improvements."*

A deceptively open brief. "Design review" and "QA check" pull in two different lenses: the first is editorial/clinical (is this the right concept, modelled the right way?), the second is normative/structural (does it pass the rules a linter would enforce?). The archetype itself — a `COMPOSITION` named *health summary* — is one of the simplest structural shapes in openEHR: a generic document container. That simplicity turned out to be the story.

## 2. The flow

I worked the openEHR Assistant **review-and-remediate pipeline** end to end:

1. **Skill routing.** The request matched the `archetype-authoring` skill (which owns the *author → review → rationale → translate* lifecycle). Its Step 7 pointed me at the `review-remediate.md` reference, whose stages are *intent & provenance → lint → remediate → review packet*.
2. **Load the rulebook.** I pulled three guides via `guide_get`: `archetypes/rules` (the 22 normative rules, grouped A–K), `archetypes/checklist` (the QA gate), and `archetypes/anti-patterns`. These define what "PASS" means and which idioms are accepted vs. flagged.
3. **Verify against the Reference Model.** Rather than trust my memory of the RM, I called `type_specification_get` for `COMPOSITION` and `EVENT_CONTEXT`, and `terminology_resolve("433")`. This confirmed: the root attributes (`category`, `context`, `content`), the `Category_validity` invariant (category must come from the `composition_category` group), and that `433` resolves to **`event`** in exactly that group. The constraint is valid.
4. **Provenance check.** The pipeline's first stage asks: *is this a mirror of a published CKM artefact?* I fetched the canonical archetype with `ckm_archetype_get("openEHR-EHR-COMPOSITION.health_summary.v1")` and diffed it against the workspace file.
5. **Synthesise.** Findings table (severity / rule / location / fix), design observations, and a checklist — with a recommendation on *where* fixes belong.

## 3. What the MCP server, plugin, and skills gave me

The value was concentrated in a few high-signal tools:

| Tool / asset | What it contributed |
|---|---|
| `archetype-authoring` skill → `review-remediate.md` | The *process*. Without it I'd have improvised an ad-hoc review; instead I followed a provenance-first pipeline that caught the single most important fact about the file. |
| `guide_get(archetypes/rules)` | A normative, citable rulebook (A1–A3 concept/scope, C2 cardinality with the `ITEM_TREE.items {0..*}` idiom exception, D2 slot constraints, E5–E7 translation accuracy, the VARID/VATDF tooling codes). Every finding maps to a rule ID. |
| `guide_get(archetypes/checklist)` + `anti-patterns` | The QA gate and the "is this open `/.*/` slot a bug?" judgement (answer: no, it's the idiomatic Extension slot). |
| `type_specification_get` (BMM) | Ground truth for RM attributes and invariants — let me assert *which* RM rule the `category` constraint satisfies, not just that it "looks right". |
| `terminology_resolve` | Deterministic confirmation that `433 = event` lives in `composition_category`. |
| `ckm_archetype_get` | **The decisive call.** It revealed the workspace file is a byte-faithful mirror of the published Foundation artefact — same `uid`, same `MD5-CAM-1.0.1`, revision `1.0.3`, `lifecycle_state = published`. |

### The finding that reframed the whole review

The provenance step turned a routine lint into a governance conversation. Once I knew the file was **canonical openEHR-Foundation content under CC-BY-SA**, every defect changed meaning:

- A genuine bug — the Swedish `at0002` label left untranslated as `"Extension"` (an **E7** translation-accuracy miss), and two English prose typos (`"containter"`, `"the all of"`) — is *also present upstream*. So the right fix is a **CKM change request**, not a local edit that would silently fork a published artefact and stale its `MD5-CAM` checksum.
- The "issues" a naive linter would scream about — the open `/.*/` include on the Extension slot, the entirely **unconstrained `content`** — are **deliberate design**, documented in the `use` text ("the main Sections/Content component has been deliberately left unconstrained"). The archetype's whole value is as a queryable container (`category = event` marker + extension area), not as a content model. Flagging them as defects would have been wrong.

Net verdict: **PASS**, with one upstream-bound WARNING, a handful of INFO notes, and two design observations (is `event` the right category for a "summary"? what does an empty-content container buy you?).

## 4. Shortcomings — what I couldn't do, and what I'd reach for next time

Honest accounting of where the toolchain left me reasoning by hand:

- **No machine validator.** The "22 normative lint rules" are applied by *me reading guide prose and eyeballing the ADL*. There is no MCP tool that runs an actual AOM validator (archie / ADL Workbench) and returns `VARID/VATDF/VACDF/VDFAI/VDFPT` plus cardinality/occurrences/existence invariant results. At-code parity across five locales, the valid-slot check, RM attribute-name correctness — all confirmed manually. A deterministic validator would be more reliable and reproducible than my read-through.
- **I could not recompute the checksum.** I asserted the `MD5-CAM` "still matches" because the content is semantically identical — but I had no tool to *recompute* it and verify. That claim is reasoned, not verified. A checksum/`build_uid` recompute tool would close the gap.
- **Provenance detection was manual.** The single most valuable insight (this is a published mirror) came from me fetching CKM and comparing `uid`/checksum by eye. This is exactly the kind of safety check that should be one tool call: *"is this local file a known CKM artefact, and how does it diverge?"*
- **No structured local↔CKM drift report.** I diffed two ADL blobs in my head. A `semantic-diff` command exists, but it isn't wired into the review pipeline, and it doesn't speak to checksum validity. Drift detection + checksum status belongs in the provenance stage as a first-class output.
- **No gold-standard COMPOSITION to lean on.** The skill suggests `examples_search(kind="archetypes")` for prior-art comparison, but the curated corpus covers OBSERVATION/EVALUATION/CLUSTER/etc. — there is **no reference COMPOSITION container**. I'd have liked a canonical "document container with Extension slot" example to compare the idiom against.
- **Severity is my judgement, not the rulebook's.** The `archetype-lint` skill advertises ERROR/WARNING/INFO per rule, but the `rules` guide doesn't encode the severity mapping. I assigned severities myself (E7 → WARNING, typos → INFO). A severity-annotated, machine-readable rules manifest would make two reviewers agree.
- **No usage/impact context.** To judge whether changing `content` or the `sv` label is *safe*, I'd want to know which workspace templates reference this archetype. That's the `/archetype-impact` command's job, but it's a separate manual step rather than an automatic note in the review packet.
- **A small guide inconsistency.** `archetype-authoring` Step 1 mandates loading `principles` + `rules` + `adl-syntax`; the `review-remediate.md` sub-reference says load `rules` + `checklist` + `anti-patterns`. I followed the more specific reference (correct for a review), but the two lists should be reconciled so the "MANDATORY" set isn't ambiguous.

## 5. Takeaways

- **Provenance-first is the right default.** For a content sandbox that mirrors CKM, "where did this file come from?" is more decisive than any lint rule. The pipeline got this ordering right, and it changed the entire recommendation.
- **Idiom literacy matters more than rule-counting.** The guides earned their keep precisely by telling me *not* to flag the open Extension slot and the unconstrained content. A linter without idiom awareness would have generated noise.
- **The biggest wins now are deterministic verification.** Most of my manual steps — validation, checksum recompute, provenance/drift — are mechanisable. They're written up as concrete proposals in [`docs/improvements/2026-06-09-openehr-assistant-tooling-improvements.md`](../improvements/2026-06-09-openehr-assistant-tooling-improvements.md).
