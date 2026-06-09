# Modelling a GGZ "Rapportage" as an openEHR template — a reuse-first session

*2026-06-09 · branch `try-2-2026-06-09` · tooling: Claude Code + openehr-assistant plugin/MCP + superpowers skills*

## The request

> "Help me create archetypes and a template for **Rapportage** — I guess it's a combination of clinical synopsis with an encounter, but I'm not sure."

A familiar shape from Dutch GGZ (mental healthcare): *Rapportage* is the ongoing
narrative reporting that care professionals write about a client. The requester had
the right intuition (clinical synopsis + encounter) but wasn't certain of the
structure — which is exactly the situation where jumping straight to authoring is a
mistake. The whole point of this write-up is the *process*, not just the output file.

## Step 0 — start with the process skill, not the code

The first action was **not** to open an editor. Per the superpowers skill-priority
rule (process skills before implementation skills), I invoked
`superpowers:brainstorming`. It enforces a hard gate: *no implementation skill, no
file authoring, until a design is presented and approved.* For a "simple" one-file
template this can feel heavy, but it paid off immediately — the design that emerged is
materially different from the one implied by the original ask.

## Step 1 — read the terrain

The workspace already carried the building blocks. A quick census of `local/` (22
`.adl`, 4 `.t.json`) surfaced three relevant archetypes:

- `openEHR-EHR-COMPOSITION.encounter.v1` — a generic contact/event wrapper
  (`category = event/433`) with just an extension slot; content is added at template level.
- `openEHR-EHR-EVALUATION.clinical_synopsis.v1` — a single free-text "Synopsis"
  field, explicitly described in its own `misuse` as **not** for structured data.
- `openEHR-EHR-EVALUATION.reason_for_encounter.v1` — available, ultimately not needed.

Reading the two candidate archetypes (not just their names) mattered: the synopsis is
a *meta-observation* — one `DV_TEXT` under `data/ITEM_TREE[at0001]/ELEMENT[at0002]`,
plus a `protocol` extension `CLUSTER[at0004]`. The encounter holds its extension slot
at `context/other_context[at0001]/items[at0002]`. Those exact node-ids became the
constraint paths later.

## Step 2 — three questions that re-shaped the design

The brainstorming skill insists on **one question at a time**. Three did the work:

1. **Which kind of GGZ rapportage?** Dagrapportage / behandelrapportage (SOEP) /
   generieke / a flexible super-model. → **Generieke rapportage.**
2. **How much structure beyond free text?** A dedicated lean `EVALUATION.rapportage`
   archetype / reuse `clinical_synopsis` as-is / a richer archetype. → **Reuse
   clinical_synopsis as-is.**
3. **Which artifact for "the template"?** OET source / canonical `.t.json` / both /
   sketch. → **OET source.**

The pivotal moment was answer #2. Reusing *both* archetypes unchanged means **there
are no new archetypes to author at all** — the entire deliverable collapses to a
single template. That directly contradicts the original "create archetypes" ask, so I
surfaced it explicitly rather than quietly proceeding. It's the openEHR *reuse-first*
principle in action: the best archetype is the one you didn't have to write.

## Step 3 — the format detour

Before committing to "produce a template", I checked *how this repo's templates are
actually built*. A short Python census of the four `.t.json` files revealed they are
**ADL-Designer-generated canonical TEMPLATE JSON** (AOM-style `rmTypeName` /
`attributes` / `nodeId`, a `t_<concept>` root specialising one parent archetype,
content-addressed `MD5-CAM` hashes, `lifecycleState = unmanaged`). Two findings shaped
the decision:

- `Vital signs.t.json` is rooted at `COMPOSITION.encounter` but is an **empty shell**
  (COMPOSITION + EVENT_CONTEXT only — *no content entries*). So the repo has **no
  worked example** of a COMPOSITION template that fills `content` with an entry
  archetype — precisely the pattern Rapportage needs.
- CLAUDE.md forbids hand-fabricating the `MD5-CAM` hashes, and those files are
  tool-generated. Hand-authoring a valid canonical `.t.json` was therefore off the table.

Hence **OET** — the hand-authorable template *source* format, which carries its own
`id`/`uid` and no content-addressed checksums.

## Step 4 — the design (approved before any code)

```
Rapportage  (template)
└─ COMPOSITION  encounter.v1            ← root (category = event)
   └─ content
      └─ EVALUATION  clinical_synopsis.v1   "Rapportage"          [min 1, max 1]
         ├─ /data[at0001]/items[at0002]     "Rapportage" free-text → min 1, max 1
         └─ /protocol[at0003]/items[at0004] extension slot         → max 0
   └─ Context
      └─ /context/other_context[at0001]/items[at0002] ext slot     → max 0
```

Crucially, *who / when / role* (composer, contact datetime, professional
participations) stays in the Reference Model / `EVENT_CONTEXT` — **not** re-modelled in
the template. Six micro-decisions (mandatory note, Dutch labels, closed extension
slots, no extra entries, identifiers) were tabled and approved before authoring.

## Step 5 — authoring the OET

I invoked `openehr-assistant:template-authoring`, whose mandatory first step is to
load the authoritative guides via the MCP `guide_get` tool. I pulled four:

- `templates/oet-syntax` — the decisive one. Its **attribute-reference table** and
  **worked snippets** (required-entry, excluded-node, rename, `textConstraint`,
  `/context/setting`, trailing `<Context>`) were directly transcribable.
- `templates/oet-idioms-cheatsheet` — the `max="0"` exclusion idiom, mandatory
  escalation, event-vs-persistent framing.
- `templates/rules` — Rule A2 (pick the COMPOSITION that fits the document type →
  `encounter`), B1/B2 (narrowing), G2 (event composition for repeated submissions).
- `templates/principles` — the narrowing principle and the CGEM framework
  (rapportage = *Event Assessment* → `category = event`).

A fresh UUID v4 (`uuidgen`-equivalent via Python) gave the `<id>`. The result is
[`local/Rapportage.oet`](../../local/Rapportage.oet).

## Step 6 — validation, and the gap behind it

There is **no automated OET/OPT validator** in this toolchain (the skill says so
outright). So I validated manually, but substantively:

- **XML well-formedness** via `ElementTree`.
- **Path resolution** — a script extracted the `at-codes` from each source archetype's
  `definition` section and confirmed every constraint path's node-ids
  (`at0001/at0002`, `at0003/at0004`, encounter `at0001/at0002`) actually exist in the
  right archetype. All resolved.
- **Narrowing legality** — only `min="1"` (mandatory) and `max="0"` (exclude); no
  expansions.

This manual check is the substantive equivalent of a linter, but it's exactly the kind
of thing a tool should do for me.

## What the openehr-assistant MCP + plugin + skills actually gave me

**Worked well:**

- The **guide corpus** is high quality and *grounded* — `oet-syntax`'s attribute table
  cites real CKM OETs and warns against inventing attributes. I authored a slot-correct
  OET almost entirely from those four guides.
- The **skill discipline** (mandatory guide-load before authoring; the narrowing
  principle repeated across `rules`/`principles`/the skill body) kept me honest.
- The **deferred-tool / ToolSearch** mechanism made the large MCP surface manageable —
  load `guide_get` and `type_specification_get` schemas only when needed.
- **superpowers:brainstorming** changed the outcome: the reuse-first, template-only
  result is better than the multi-archetype build the prompt implied.

**Loaded but unused:** `type_specification_get` (I never needed to inspect a BMM class —
the archetypes told me everything). Untouched entirely: `ckm_archetype_search` /
`ckm_template_search`, `examples_search/get`, `terminology_resolve`.

## Shortcomings, misses, and a wishlist

1. **No template validator.** The single biggest gap. I want a `template_validate`
   MCP tool that loads an OET, resolves every `<Rule path>` against the referenced
   archetypes, and flags illegal narrowings (e.g. `min` below the archetype minimum,
   `max` above its maximum, paths that don't exist). I rebuilt a weak version of this
   in throwaway Python.
2. **No template example with content.** `examples_search` covers
   `aql | flat | structured | archetypes` — but **not templates**. The one structural
   pattern I needed (a COMPOSITION root filling `content` with an entry) had no worked
   example anywhere — local repo or examples corpus. I'd add an
   `openehr://examples/templates/` namespace with at least one real, content-bearing OET.
3. **Under-specified `<description>` serialization.** `oet-syntax` describes the
   *contents* of `<description>` (lifecycle/purpose/use/misuse/other_details) but not
   the exact child-element XML. I had to ground it by convention and flag it as
   unverified. A complete canonical OET example would close this.
4. **`<Content>` occurrence attributes are ambiguous.** The attribute table lists `max`
   on `<Content>` but not `min`; I used `min="1"` (documented on sibling placement
   elements, and used by real Ocean OETs). The table should say which occurrence
   attributes are valid where.
5. **Reuse survey shortcut.** Strict reuse-first says *search CKM before authoring*. I
   leaned on the already-present local archetypes (which the user had named) instead of
   running `ckm-scout` / `ckm_archetype_search`. Defensible here, but worth flagging.
6. **Process friction for small artifacts.** The generic brainstorming → writing-plans
   → executing-plans flow is calibrated for multi-step software features; for a
   single-file OET it's heavy, and the user short-circuited it with "go ahead". A
   lighter openEHR-authoring path would fit these tasks better.
7. **No OET → `.t.json` / OPT bridge.** Because `.t.json` carries un-fabricatable MD5
   hashes, the OET I produced can't become the repo's runtime artifact without external
   tooling. A conversion/checksum helper would remove that dead-end.

## Takeaways

- The reuse-first answer is often "you don't need a new archetype." Surfacing that
  early — even when it contradicts the literal request — is the job.
- Read archetypes, don't just name them; the node-ids *are* the template.
- The openehr-assistant guides are strong enough to author from directly; the missing
  piece is **verification tooling**, not knowledge.

*Companion refactor brief:* [`docs/improvements/2026-06-09-openehr-assistant-template-authoring-improvements.md`](../improvements/2026-06-09-openehr-assistant-template-authoring-improvements.md)
