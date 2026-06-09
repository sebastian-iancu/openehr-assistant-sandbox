# Modelling a Six-Minute Walk Test — a reuse-first, subagent-driven build

*2026-06-09 · branch `try-2-2026-06-09` · tooling: Claude Code + openehr-assistant plugin/MCP + superpowers skills*

## The request

> "Use your openEHR skills and help me out to model a 6-minute walk test set of template and associated archetypes."

Unlike a "review this" or "translate that" task, this one asked for **net-new
modelling**: an archetype *and* a template, for a well-known clinical instrument. The
6-Minute Walk Test (6MWT; ATS 2002, ATS/ERS 2014) is a sub-maximal exercise test —
the patient walks as far as possible in six minutes, and the primary outcome is the
**6-minute walk distance (6MWD)** in metres, read alongside heart rate, blood pressure,
SpO₂ and perceived exertion before and after the walk. The interesting modelling
question isn't "what is a 6MWT" — it's *how much to build versus reuse*.

## Step 0 — process skill before editor

Per the superpowers skill-priority rule (process skills before implementation skills),
the first action was `superpowers:brainstorming`, which hard-gates any authoring until
a design is approved. For a build-from-scratch request this is clearly the right call —
the cost of a wrong structure compounds across an archetype *and* a template.

## Step 1 — a reuse-first CKM survey (the part that shapes everything)

openEHR's cardinal rule is **reuse before authoring**, so before designing anything I
ran a survey of the Clinical Knowledge Manager via the MCP `ckm_archetype_search` tool.
This is where the design got its skeleton:

| Building block | CKM archetype | State | Verdict |
|---|---|---|---|
| Heart rate / pulse | `OBSERVATION.pulse.v2` | PUBLISHED | REUSE |
| Blood pressure | `OBSERVATION.blood_pressure.v2` | PUBLISHED | REUSE |
| Pulse oximetry / SpO₂ | `OBSERVATION.pulse_oximetry.v1` | PUBLISHED | REUSE |
| Supplemental O₂ | `CLUSTER.inspired_oxygen.v1` | PUBLISHED | REUSE (slot) |
| Borg / exertion | `CLUSTER.level_of_exertion.v0` | **DRAFT (alpha)** | avoid — embed instead |
| Composition root | `COMPOSITION.report-result.v1` | PUBLISHED | REUSE (root) |
| **6MWT itself** | *none* (closest: `OBSERVATION.timed_25_foot_walk.v1`) | — | **NEW** |

The decisive finding: **there is no published 6MWT OBSERVATION in CKM**, which
*justifies* authoring a new one — and the published `timed_25_foot_walk.v1` (a timed
functional walk test) gave a precedent structure to copy the ADL idiom from. Equally
important, the canonical Borg cluster `level_of_exertion` is only **draft-alpha**, so
depending on it would have built the model on shifting ground.

(Two friction notes up front: the very first tool calls failed because the workspace
had no MCP permission allow-list — I had to create `.claude/settings.json` to permit
the `mcp__…openehr-assistant` tools. And the dedicated `ckm-scout` agent stalled —
it narrated search intentions repeatedly and returned **0 tool calls**, twice — so I
ran `ckm_archetype_search` directly instead. Both reappear in the wishlist.)

## Step 2 — scoping decisions, batched

The user is an openEHR expert, so rather than the brainstorming default of one question
at a time, I used `AskUserQuestion` to settle the genuine forks in two rounds. The
answers:

- **Reuse:** ground the design on a CKM survey (done above).
- **Architecture:** a COMPOSITION-rooted template aggregating a dedicated 6MWT
  OBSERVATION **plus** the reused vital-sign observations — the most RM-idiomatic shape.
- **Timing:** a **single summary event** inside the 6MWT OBSERVATION (distance, nadir
  SpO₂, peak HR, end-of-test Borg) — the discrete resting/recovery readings live in the
  reused vitals.
- **Ancillary data:** supplemental O₂ only (predicted distance, stops/rests, course
  metadata deliberately deferred).
- **Borg:** **embed** two `DV_ORDINAL` 0–10 ratings rather than slot the draft cluster.
- **Root:** `COMPOSITION.report-result.v1` (precise for a test report).
- **Template output:** a structurally valid `.t.json` with the tool-generated
  `MD5-CAM`/`buildUid` hashes left as **flagged placeholders** for Archetype Designer to
  regenerate (CLAUDE.md forbids hand-fabricating them).

## Step 3 — the design, written down and approved

The deliverable collapsed to **two new files**, everything else referenced by id:

```
Six minute walk test (template)
└─ COMPOSITION  report-result.v1                       ← root
   └─ content
      ├─ OBSERVATION  six_minute_walk_test.v0   [1..1]  ← NEW
      ├─ OBSERVATION  pulse.v2                  [0..1]  ← reused
      ├─ OBSERVATION  blood_pressure.v2         [0..1]  ← reused
      └─ OBSERVATION  pulse_oximetry.v1         [0..1]  ← reused

OBSERVATION six_minute_walk_test.v0 — single EVENT[at0002] "6MWT summary"
  data:  distance(1..1, m) · laps · lowest SpO₂(%) · peak HR(/min)
         · Borg dyspnoea(ordinal 0–10) · Borg fatigue(ordinal 0–10)
         · test completed · interpretation · comment
  state: slot → CLUSTER.inspired_oxygen.v1
  protocol: open Extension slot
```

The design was written to `docs/superpowers/specs/`, self-reviewed, and approved before
a single line of ADL was authored.

## Step 4 — plan, then subagent-driven execution

I wrote a task-by-task plan (`writing-plans`), adapting the TDD rhythm to the domain:
the "test" at each checkpoint is **`archetype-lint` STRICT = 0 errors** and structural
JSON validation, not pytest. Execution used `subagent-driven-development`: one
`openehr-assistant:clinical-modeler` implementer per *artifact* (the natural unit, since
all tasks edit the same two files), with verification gates between.

The implementers did good work, but the **most valuable moments were the review gates**:

1. **The archetype implementer flagged that it could not reach MCP.** Its agent
   definition lists `terminology_resolve` and `ckm_archetype_get`, but at runtime those
   were unavailable to the subagent, so it *inferred* openEHR property codes from on-disk
   archetype copies. Back in the main session — which **does** have MCP — I verified the
   codes with `terminology_resolve`:
   - `122` → **Length** ✓ (distance in m)
   - `382` → **Frequency** ✓ (heart rate /min — matches canonical `pulse.v2`)
   - `380` → **Qualified real** ✓ (percentage `DV_QUANTITY` for SpO₂)

   This also caught **my own plan's errors**: I'd suggested `125` (which resolves to
   *Pressure*) for the percentage and `384` (*Amount of substance*) for the rate. The
   implementer's inferred codes were *better than my plan's*, and `terminology_resolve`
   settled it authoritatively. Grounding beat both guesses.

2. **STRICT lint passed clean** (0 errors / 0 warnings). Running `archetype-lint` meant
   loading its mandatory guides (`archetypes/rules`, `archetypes/structural-constraints`)
   — and those guides pre-empt the classic false positive: a mandatory `Distance walked`
   inside an `items {0..*}` container is the *idiom* (citing `ecg_result.v1`), not a
   violation. That note alone saved a wrong "everything should be optional" finding.

3. **The template implementer caught a bug in my plan**, not its own work: my
   cross-reference `grep` used the character class `[a-z_]`/`[a-z_0-9]`, which excludes
   the hyphen in `report-result` and would falsely report a missing reference. The file
   was correct; the verification command was wrong. I fixed the plan.

The branch was left as-is at the user's discretion (it also carries unrelated parallel
work), with the template's hashes explicitly marked `REGENERATE_IN_ARCHETYPE_DESIGNER`.

## What the openehr-assistant MCP + plugin + skills gave me

**Worked well:**

- **`ckm_archetype_search`** was fast and decisive — it both *found* the reuse blocks
  (pulse/BP/SpO₂/inspired_oxygen/report-result) and, by *not* finding a 6MWT, justified
  the new archetype. Reuse-first is only credible when the search is real.
- **`terminology_resolve`** is the quiet hero. A rule-based lint can't tell you whether
  `openehr::382` is the right property for `/min`; this tool can, and it overruled two
  bad guesses (mine and the subagent's). It is the single best grounding tool here.
- **The lint guide corpus** is authoritative *and* anticipates false positives — the
  `items {0..*}`-with-mandatory-element carve-out is exactly the nuance a naive linter
  gets wrong.
- **The skill chain** (`brainstorming → writing-plans → subagent-driven-development →
  finishing-a-development-branch`) gave the work a spine, and the `clinical-modeler`
  agent kept the heavy ADL/JSON editing out of the main context.

**Used indirectly / not needed:** `ckm_archetype_get` (I relied on the precedent only
conceptually), `type_specification_get`, `examples_search` (and see below — it isn't
reachable from `clinical-modeler` anyway).

## Shortcomings, misses, and a wishlist

1. **Subagent MCP access is broken where it matters most.** The `clinical-modeler`
   agent is *designed* to ground itself via `terminology_resolve` / `ckm_archetype_get`,
   but those tools weren't actually available to it at runtime — so the one agent built
   for grounded authoring had to *guess* property codes from disk. This is the highest-
   impact finding of the session.
2. **No real ADL/AOM parser.** `archetype-lint` is rule-based heuristics, not a parse.
   It can't confirm true ADL 1.4 syntactic validity — e.g. whether `DV_BOOLEAN`
   constraints should be `true,false` vs `True,False`, or that a `DV_ORDINAL`
   `n|[local::atNNNN]` list parses. I'd want an Archie-backed `archetype_parse`/validate.
3. **No `.t.json` (web template / CAM) validator or builder, and no content-bearing
   example.** The repo's `Vital signs.t.json` is an empty `encounter` shell, so there
   was **no worked example** of a COMPOSITION template that fills `content` with
   `C_ARCHETYPE_ROOT` entries — the exact pattern I needed. I had to hand-write the
   `content` JSON skeleton into the implementer's prompt. And the `MD5-CAM` hashes can't
   be computed offline, so the template is structurally valid but not deployable without
   Archetype Designer.
4. **`examples_search` has no `templates` namespace and isn't in `clinical-modeler`'s
   toolset.** My plan told the implementer to look up the `C_ARCHETYPE_ROOT` shape via
   examples; it couldn't. (This echoes the prior Rapportage session's finding, now with
   a second concrete case — the CAM JSON format, not just OET.)
5. **Property-code lookup is a manual leap.** Getting from "SpO₂ in %" to
   `property=[openehr::380]` required either reading another archetype or resolving
   codes one by one. A units→property idiom table (length=122/m, frequency=382//min,
   qualified-real=380/%) in the ADL idioms cheatsheet would prevent the guesswork that
   bit both my plan and the subagent.
6. **`ckm-scout` reliability.** The dedicated reuse-survey agent stalled twice, returning
   zero tool calls; I fell back to calling `ckm_archetype_search` myself. A specialized
   agent that doesn't issue its tool is worse than no agent.
7. **MCP permission bootstrap.** A fresh content workspace had no allow-list, so the
   first MCP calls failed outright. The plugin ships its own `.claude/settings.json`;
   the companion content repo should too (or the plugin should advertise the required
   permissions).
8. **DV_SCALE vs ADL 1.4.** The modified-Borg 0.5 rung can't be represented by an
   integer `DV_ORDINAL`, and `DV_SCALE` needs RM ≥ 1.1.0. An accepted limitation here,
   but a guide note on "non-integer rating scales → DV_SCALE + RM-version implications"
   would make the trade-off explicit at design time.

## Takeaways

- **Reuse-first is a search, not a slogan.** The CKM survey both supplied the skeleton
  *and* licensed the one new archetype by proving nothing equivalent exists.
- **Grounding tools earn their keep at the leaves.** `terminology_resolve` corrected two
  independent property-code guesses — the kind of error a rule-based linter never sees.
- **The gap is verification and grounding *reach*, not knowledge.** The guides are
  strong enough to author from; what's missing is a real parser, a template validator,
  a content-bearing template example, and — most of all — letting the authoring subagent
  actually reach the grounding tools it was given.

*Companion refactor brief:*
[`docs/improvements/2026-06-09-openehr-assistant-subagent-and-validation-improvements.md`](../improvements/2026-06-09-openehr-assistant-subagent-and-validation-improvements.md)
