# Modelling a Dutch GGZ "Rapportage" with the openEHR Assistant — a session retrospective

*2026-06-09 · a working log of how an AI pair-modeller took an admittedly-uncertain "help me model a Rapportage… I think?" from a one-line request to a single committed, validated OET — by first figuring out what the clinician actually meant, and where the toolchain (and one permission wall) got in the way.*

---

## The request

> *"I would like you to help creating a template for what in dutch landscape is called Rapportage — I guess is a combination of clinical synopsis with an encounter — but I am not sure."*

The honesty in "but I am not sure" *is* the problem statement. *Rapportage* is one of those words that means something precise to a Dutch clinician and almost nothing portable to an information model. Depending on the setting it is: daily caregiver reporting in long-term care (VVT/ECD, often SOEP-structured and tied to a care plan); a curative contact/progress note (decursus); or mental-health reporting. Each maps to a different openEHR shape. So the first deliverable was never ADL — it was a *disambiguation*. Get that wrong and you build a beautifully-linted artifact that models the wrong thing.

The user's own hypothesis — *clinical synopsis + encounter* — turned out to be exactly right. But confirming that, and pinning the rest, was the work.

## How it was solved

The session followed a deliberate pipeline; almost no time was spent in ADL.

1. **Brainstorming first — one decision at a time.** Rather than guess, the highest-leverage forks were put to the user as structured choices, sequentially:
   - *Setting/purpose* → **GGZ decursus** (mental-health treatment reporting). This single answer determined everything downstream.
   - *Body structure* → **narrative + optional structure** (free text first, SOEP optional).
   - *Treatment-goal (behandeldoel) linkage* → **skip for v1** (YAGNI).
   - *Realisation approach* → **A: lean reuse, zero new archetypes** (SOEP as a writing convention).
   - *Two honest tensions surfaced and resolved:* coded contact type/modality has no published holder, so **deferred to v2**; and the deliverable, initially "both OET + .t.json", was narrowed by the user mid-flight to **OET only**.

   Each answer *shrank* the scope. The arc went from "a Rapportage template" to "one event `COMPOSITION.encounter` containing a mandatory narrative and an optional reason — one file."

2. **Reuse-first survey (with a detour — see Shortcomings).** The most-violated openEHR principle is *reuse before authoring*. A context-isolated `ckm-scout` agent was dispatched to survey CKM… and came back **blocked** — it had no CKM access at all. The survey was re-run **inline from the main session**, which *did* have the tools. Headline findings: **no dedicated international "progress note / decursus" template or archetype exists**, but the building blocks are mature and PUBLISHED — `COMPOSITION.encounter.v1` (rev 1.0.12), `EVALUATION.clinical_synopsis.v1` (rev 1.0.4, the top hit for "clinical synopsis"), `EVALUATION.reason_for_encounter.v1`, and `EVALUATION.goal.v1` for the deferred behandeldoel. The decisive insight: the **international CKM only carries generic scaffolding; a SOEP-structured / behandeldoel / contact-type model belongs to the Dutch national CKM / Nictiz zib space**, which is out of v1 scope (and out of the toolchain's reach).

3. **Grounding the facts, not guessing them.** `terminology_resolve` confirmed care `setting` = *mental healthcare* → **`openehr::802`**, and the composition `category` *event* → **`openehr::433`** (matching the encounter archetype). The `clinical_synopsis` narrative element was confirmed at path `/data[at0001]/items[at0002]`.

4. **Grounding the *format* — and a surprise.** The plan was to mirror the repo's existing `.t.json` templates. Reading them revealed they are **not** the Better/EHRbase "web template" I assumed — they are **ADL Designer differential-template JSON**, carrying tool-generated `MD5-CAM` and `buildUid` checksums that cannot be hand-authored. This is precisely why the user pivoted to **OET only**. To author the OET correctly, a **real CKM OET was fetched** (`ckm_template_get`, format `oet`) to lock down the exact XML schema — `<definition xsi:type="COMPOSITION" archetype_id=… concept_name=…>` → `<Content xsi:type="EVALUATION" … path="/content">` → nested `<Rule>`, trailing `<Context/>`, and tool-generated `<integrity_checks>` digests (deliberately **omitted** rather than fabricated). That same real OET also revealed a published `SECTION.soap.v1` — a reuse path for SOEP that the keyword search had missed.

5. **Spec → plan → subagent-driven execution.** A short design spec captured the decisions (and was revised in place when the deliverable narrowed to OET-only); a plan turned it into one bite-sized task with the literal file content and `python3` + `/template-explain` as the validation gates. A fresh implementer subagent wrote and committed the file; a spec-compliance reviewer and a code-quality reviewer then independently verified it.

The result, committed to `main`:

| Artifact | What it holds |
|---|---|
| `local/GGZ Rapportage.oet` | `COMPOSITION.encounter` (event) → mandatory `EVALUATION.clinical_synopsis` ("Rapportage", `Synopsis` 1..1) + optional `EVALUATION.reason_for_encounter` ("Reden van contact", 0..1); `setting` 802 documented; reporter via RM `composer`/participations; zero new archetypes |

Validation: XML well-formed, structural assertions pass, `/template-explain` clean, both reviews approved (no required changes).

## What the openEHR Assistant MCP, plugin, and skills actually contributed

The domain work was tool-grounded, not recalled:

- **`ckm_archetype_search` / `ckm_template_search` / `ckm_template_get`.** The reuse survey (run inline after the agent was blocked) and — crucially — a *real OET* fetched as a structural exemplar. Fetching a known-good artifact beat reconstructing the OET schema from memory.
- **`terminology_resolve`.** `setting::802` (mental healthcare) and `category::433` (event), verified instead of guessed — the kind of code that silently corrupts a `DV_CODED_TEXT` when fabricated.
- **`guide_get(templates/oet-syntax)`.** The technical description of the OET XML format (root element, `Content`/`Items`/`Rule`, constraint types) — the backbone of the authored file.
- **`/template-explain` skill.** Parsed and explained the finished OET (root, both contents, cardinalities) as the semantic validation gate.
- **`ckm-scout` agent** — *intended* to own the reuse survey (it was blocked; see below).
- **Superpowers process skills** (`brainstorming`, `writing-plans`, `subagent-driven-development`, `finishing-a-development-branch`) — the scaffolding that forced disambiguation *before* code and review *after* it.

As with the sibling 6MWT session, the division of labour is the point: the **plugin/skills carried openEHR domain knowledge**, the **superpowers skills carried engineering discipline**. Here the brainstorming half did the heaviest lifting, because the request's real difficulty was semantic, not technical.

## Shortcomings and gaps — what was missing or awkward

1. **The reuse-first agent was blocked — the flagship workflow failed where it should shine.** `ckm-scout` is *the* agent for reuse surveys, yet when dispatched (in the background) it had **every CKM path denied** — both MCP namespaces, `curl`, and `WebFetch`. The identical tools worked fine **from the main session moments later**. This main-session-vs-subagent capability asymmetry is the session's biggest friction: the context-isolated survey (the whole reason `ckm-scout` exists) had to be abandoned and re-done inline, polluting the main context with exactly the search output the agent was meant to absorb. **Background/subagents need the same MCP access the main loop has**, or the reuse-first pattern is theatre.

2. **No Dutch national CKM / Nictiz zib access — and this was a *Dutch* task.** The genuinely relevant artefacts for a GGZ Rapportage — SOEP structures, behandeldoel value sets, contact-type/ZPM registration codes — live in the **NL national space**, which the toolchain cannot search. For non-international modelling this is the deepest gap: the assistant can only confirm that the right answer is "somewhere you can't look."

3. **The repo's template serialization was undocumented and surprising.** The existing `.t.json` files are ADL Designer differential JSON, not web templates — discovered only by reading them. No guide describes *this repo's* template conventions, and (as in 6MWT) **there is no template compiler / OPT builder and no `MD5-CAM`/`buildUid` generator** in the MCP. That missing capability is the direct cause of the user dropping `.t.json` entirely. A `template_build` / `opt_compile` tool remains the single highest-value thing to add.

4. **No authoritative OET/ADL validator.** `/template-explain` is an LLM applying knowledge, not an AOM/OET schema parser. I could not get a *machine* answer to "is `min` legal on a `<Content>`? where exactly does a `/context/setting` constraint go?" The workaround was to fetch a real OET and pattern-match — and, where uncertainty remained (fixing `setting` to a single code), to **downgrade the constraint to documentation** rather than risk an invalid file. An Archie-backed `oet_validate` would have let me actually *constrain* setting to 802 with confidence instead of just describing it.

5. **Archetype-search recall missed an on-point reuse.** A direct search for SOAP/SOEP archetypes returned nothing, yet `SECTION.soap.v1` plainly exists — I only stumbled on it inside an unrelated demo OET. For a reuse-first methodology, a search that misses a published, exactly-relevant archetype is a real recall problem.

6. **The dual MCP namespace is confusing, and one half dropped mid-session.** Tools were exposed under both `plugin_openehr-assistant_…` and `claude_ai_openehr-assistant_…`; the `claude_ai` server **disconnected partway through**. Everything kept working via the plugin namespace, but the redundancy is a foot-gun — it's not obvious which to call, and a blocked/disconnected one looks like a capability gap rather than a routing issue.

7. **The authoring agent is walled off from the knowledge tools (again).** `clinical-modeler` can write files but has no MCP/CKM access; the implementer therefore had to be a `general-purpose` agent just so it could run `/template-explain`. Every grounded fact had to be hand-carried into the prompt. A modelling agent that can both look up and write would end this context-shuffling tax.

8. **`uuidgen` absent.** A papercut (worked around via `python3 -c 'import uuid'`), same family of environment friction as the sibling session.

## What I'd reach for next time

- **Subagents with parity MCP access** — so `ckm-scout` and `clinical-modeler` actually work where they're meant to.
- **NL national CKM / Nictiz zib search** — the missing half of any Dutch modelling task.
- A **`template_build` / OPT-compile tool** (+ hash generation), so a template stops being a stub.
- An **Archie-backed `oet_validate` / `adl_validate`** for parse-level truth, complementing the semantic lint/explain.
- **Better archetype-search recall** (the `SECTION.soap.v1` miss).
- A **modelling agent with both file-write and MCP-lookup**, ending the prompt hand-off.
- A short **"this repo's template conventions" guide** so the `.t.json`-vs-`.oet` surprise isn't rediscovered each time.

## Takeaways

- **Disambiguation *is* the model.** The single most valuable act was turning "Rapportage… I think?" into "GGZ decursus." Once the clinical concept was pinned, the openEHR mapping (encounter + clinical_synopsis) fell straight out — and confirmed the user's own instinct.
- **Reuse-first doubled as a reality check.** It validated that nothing needed inventing internationally, *and* correctly located the Dutch-specific parts in a national space that's out of scope (and out of reach) — which is itself a useful, honest result.
- **Scope discipline compounded.** Every user decision — skip goal linkage, approach A, defer contact type, drop `.t.json` — removed work. The session is a small case study in subtraction: one clean OET that can *grow* into the SOEP/behandeldoel v2, rather than a sprawl that has to be redone.
- **Grounded facts overturned my assumptions.** The `.t.json` format, the OET schema, the existence of `SECTION.soap.v1`, codes 802/433 — several of my initial beliefs were wrong and only verification caught them.
- **The last mile is unchanged from 6MWT.** Authoring a correct design-time artifact is well-supported; turning it into a deployable operational template still escapes the toolchain. That, plus the subagent permission wall, are the two things most worth fixing.

---

*Artifacts from this session:* `local/GGZ Rapportage.oet`, with the design spec (`docs/superpowers/specs/2026-06-08-ggz-rapportage-template-design.md`) and implementation plan (`docs/superpowers/plans/2026-06-09-ggz-rapportage-template.md`).
