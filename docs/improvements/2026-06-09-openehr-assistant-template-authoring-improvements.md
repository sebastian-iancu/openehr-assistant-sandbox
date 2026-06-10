# Refactor brief — openEHR Assistant template-authoring improvements

**Audience:** a refactor AI agent (or maintainer) working on the
`openehr-assistant-mcp` server and the `openehr-assistant-plugin` Claude Code plugin.
**Source:** findings from a real authoring session (2026-06-09) that produced
`local/Rapportage.oet` (GGZ generic *Rapportage* = `COMPOSITION.encounter` +
`EVALUATION.clinical_synopsis`). See the companion retrospective in
[`docs/blog/2026-06-09-modelling-ggz-rapportage-as-an-openehr-template.md`](../blog/2026-06-09-modelling-ggz-rapportage-as-an-openehr-template.md).

## How to use this brief

Each item has: **Repo · Problem · Evidence · Proposed fix · Acceptance test ·
Priority (P0–P2) · Effort**. Do not bundle unrelated changes. Verify each fix against
its acceptance test. Where a fix spans both repos (a new MCP capability plus a plugin
skill/command that calls it), implement the MCP side first.

Repos referenced:
- **MCP** = `openehr-assistant-mcp` (the server: `#[McpTool]`s, `resources/guides/`,
  `resources/examples/`, `resources/prompts/`).
- **PLUGIN** = `openehr-assistant-plugin` (skills, commands, agents consumed by Claude Code).

---

## P0 — high impact, fills a capability gap

### 1. Add a `template_validate` MCP tool
- **Repo:** MCP (+ PLUGIN command/skill wrapper)
- **Problem:** There is **no automated OET/OPT validator**. Authors (human or agent)
  must hand-check that constraint paths exist and that narrowing is legal.
- **Evidence:** The `template-authoring` skill states "There is no automated OET/OPT
  validator available"; this session validated `Rapportage.oet` with throwaway Python
  (`ElementTree` parse + at-code extraction from each archetype's `definition` section
  + manual narrowing check). That logic should live in the server.
- **Proposed fix:** New tool `template_validate(oet_or_opt, {resolve_against:
  local|ckm})` that:
  1. parses the OET/OPT,
  2. for every `<Content>`/`<Item>`/`<Items>` placement, loads the referenced archetype
     and confirms each `<Rule path>` resolves to a real node (`at`/`ac`-code path),
  3. checks each occurrence override is a legal **narrowing** (`min ≥ archetype min`,
     `max ≤ archetype max`, never `min > max`),
  4. checks `textConstraint`/`quantityConstraint` only subset the archetype's value set
     / unit range,
  5. returns a structured report (ERROR/WARNING/INFO) mirroring `archetype-lint`.
- **Acceptance test:** Running it on `local/Rapportage.oet` returns 0 errors; mutating
  one path to `items[at9999]` yields an ERROR; setting the synopsis `min="2"` (above
  the element's max) yields a narrowing ERROR.
- **Priority:** P0 · **Effort:** M–L

### 2. Add a `/template-lint` command + skill in the plugin
- **Repo:** PLUGIN (depends on item 1)
- **Problem:** There is an `archetype-lint` skill/command but **no template
  equivalent**. The `template-authoring` skill embeds a Step-9 checklist that is run by
  hand.
- **Proposed fix:** A `/template-lint <file>` command and a thin `template-lint` skill
  that call `template_validate` and render the report; mirror the STRICT/PERMISSIVE
  mode split of `archetype-lint`.
- **Acceptance test:** `/template-lint local/Rapportage.oet` produces a clean report;
  the skill description triggers on "lint/validate a template".
- **Priority:** P0 · **Effort:** S

### 3. Add a template examples namespace with a content-bearing OET
- **Repo:** MCP
- **Problem:** `examples_search`/`examples_get` cover `aql | flat | structured |
  archetypes` — **not templates**. The one structural pattern needed this session — a
  COMPOSITION root that fills `content` with an ENTRY archetype — had **no worked
  example** anywhere (the repo's `Vital signs.t.json` is an empty encounter shell).
- **Proposed fix:** Add `openehr://examples/templates/{name}` and seed it with at least
  one real OET that composes a COMPOSITION + an ENTRY (e.g. `encounter` +
  `clinical_synopsis`, i.e. this session's `Rapportage.oet`), with a metadata header
  describing the pattern and the narrowing rules it demonstrates.
- **Acceptance test:** `examples_search("encounter narrative template")` returns the
  new entry; `examples_get` yields a slot-correct OET.
- **Priority:** P0 · **Effort:** S–M

---

## P1 — correctness/clarity of existing guidance

### 4. Complete the `<description>` serialization in `templates/oet-syntax`
- **Repo:** MCP (`resources/guides/templates/oet-syntax`)
- **Problem:** The guide lists what `<description>` *contains* (lifecycle_state /
  purpose / use / misuse / other_details) but not the exact child-element XML. The OET
  produced this session had to ground the `<description>` block **by convention** and
  flag it as unverified.
- **Proposed fix:** Add a complete, copy-pasteable `<description>` block (showing
  `original_author` key/value items, `lifecycle_state`, `purpose`/`use`/`misuse`, and
  `other_details` with `original_language`), ideally as part of one full canonical OET
  example in the guide.
- **Acceptance test:** A reader can author a `<description>` without guessing element
  names; the example round-trips through Archetype/Template Designer unchanged.
- **Priority:** P1 · **Effort:** S

### 5. Disambiguate occurrence attributes on `<Content>` in the attribute table
- **Repo:** MCP (`resources/guides/templates/oet-syntax`)
- **Problem:** The attribute-reference table lists `max` on `<Content>` but not `min`,
  while `<Item>`/`<Items>`/`<Rule>` list both. This forced a judgement call
  (`min="1"` on the `Content` for the mandatory single rapportage entry).
- **Proposed fix:** State explicitly that `min` and `max` are both valid occurrence
  overrides on `<Content>`/`<Item>`/`<Items>`, or document the canonical way to make a
  content entry mandatory-single.
- **Acceptance test:** The table answers "how do I require exactly one entry?" without
  inference.
- **Priority:** P1 · **Effort:** XS

### 6. Make the "reuse-first may yield a template-only deliverable" outcome explicit
- **Repo:** PLUGIN (`template-authoring` and/or `archetype-authoring` skill)
- **Problem:** A request to "create archetypes and a template" can legitimately resolve
  to **no new archetypes** once reuse-first is applied. This session derived that
  insight manually and had to surface the contradiction to the user.
- **Proposed fix:** Add a short note to the template-authoring skill: *before authoring,
  confirm whether reuse of published archetypes makes new archetypes unnecessary, and
  surface that to the user explicitly.* Cross-link to the reuse-first / `ckm-scout`
  guidance.
- **Acceptance test:** The skill text prompts the agent to check reuse-first and report
  a template-only outcome when applicable.
- **Priority:** P1 · **Effort:** XS

---

## P2 — workflow smoothing

### 7. Provide an archetype-path helper
- **Repo:** MCP
- **Problem:** Authors derive constraint paths (`/data[at0001]/items[at0002]`) by hand
  from archetype node-ids — error-prone and the main thing `template_validate` later
  re-checks.
- **Proposed fix:** A `archetype_paths_get(archetype_id)` tool returning the valid
  constrainable paths (with node text) for an archetype, so OET `<Rule path>` values can
  be selected rather than transcribed. (Could be folded into an existing explain tool.)
- **Acceptance test:** For `clinical_synopsis.v1` it lists `/data[at0001]/items[at0002]`
  ("Synopsis") and `/protocol[at0003]/items[at0004]` ("Extension").
- **Priority:** P2 · **Effort:** M

### 8. Document/automate the OET → `.t.json` / OPT bridge
- **Repo:** MCP (+ docs)
- **Problem:** The repo's runtime templates are `.t.json` with content-addressed
  `MD5-CAM` hashes that CLAUDE.md forbids hand-fabricating. A hand-authored OET cannot
  become the runtime artifact without external tooling, leaving a dead-end.
- **Proposed fix:** Either (a) document the supported external toolchain (ADL/Template
  Designer) and where each checksum comes from in `templates/serialization-formats`, or
  (b) provide a conversion/checksum helper tool. At minimum, make the dead-end explicit
  so agents stop at OET deliberately.
- **Acceptance test:** A reader knows exactly how `Rapportage.oet` becomes a deployable
  `.t.json`/OPT and which checksums are tool-computed.
- **Priority:** P2 · **Effort:** S (docs) / L (tooling)

### 9. Offer a lightweight authoring path for single-file artifacts
- **Repo:** PLUGIN (process docs / skill interplay)
- **Problem:** The generic `brainstorming → writing-plans → executing-plans` flow is
  calibrated for multi-step features; for a single OET it is heavy, and the user
  short-circuited it ("go ahead").
- **Proposed fix:** Document that for a confirmed, single-artifact openEHR authoring
  task, `template-authoring`/`archetype-authoring` can be entered directly after design
  approval without a full implementation plan.
- **Acceptance test:** The skill guidance distinguishes "single artifact" from
  "multi-step plan" and routes accordingly.
- **Priority:** P2 · **Effort:** XS

---

## Summary matrix

| # | Title | Repo | Priority | Effort |
|---|-------|------|----------|--------|
| 1 | `template_validate` MCP tool | MCP | P0 | M–L |
| 2 | `/template-lint` command + skill | PLUGIN | P0 | S |
| 3 | `examples/templates/` namespace (content-bearing OET) | MCP | P0 | S–M |
| 4 | Complete `<description>` serialization in oet-syntax | MCP | P1 | S |
| 5 | Disambiguate `<Content>` occurrence attributes | MCP | P1 | XS |
| 6 | Surface "template-only / no new archetypes" outcome | PLUGIN | P1 | XS |
| 7 | `archetype_paths_get` helper | MCP | P2 | M |
| 8 | OET → `.t.json`/OPT bridge (docs/tooling) | MCP | P2 | S/L |
| 9 | Lightweight path for single-file authoring | PLUGIN | P2 | XS |
