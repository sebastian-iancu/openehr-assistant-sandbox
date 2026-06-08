# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **content-only sandbox**: demo openEHR archetypes and templates for the `openehr-assistant-mcp` server. There is no application source code, no build system, and no test/lint tooling. "Working in this repo" means authoring, editing, and validating clinical modelling artifacts — not compiling software.

All artifacts live in `local/`:

- **Archetypes** — `*.adl` files in ADL 1.4 (`adl_version=1.4`). Filenames follow the openEHR convention `openEHR-EHR-<RM_CLASS>.<concept>.<version>.adl`, e.g. `openEHR-EHR-OBSERVATION.laboratory_test_result.v1.adl`. RM classes present: CLUSTER (the bulk — anatomical/exam models), OBSERVATION, EVALUATION, COMPOSITION. Versions are `v0` (draft) through `v2`.
- **Templates** — `*.t.json` files in openEHR Web Template (simplified) JSON format, with a top-level `"@type": "TEMPLATE"`. These compose archetypes into usable forms (e.g. `Vital signs.t.json`).

## Authoring conventions

- Match the structure of existing files exactly. ADL files are whitespace/indentation sensitive (tabs) and have a fixed section order: `archetype` header → `concept` → `language` (with `translations`) → `description` → `definition` → `ontology`/`terminology`. Node IDs use the `[atNNNN]` / `[acNNNN]` pattern.
- Templates carry MD5 hashes in `otherDetails` (`MD5-CAM-1.0.1`, `PARENT:MD5-CAM-1.0.1`) and a `uid`. Preserve existing hashes/UIDs unless intentionally regenerating; do not hand-fabricate them.
- New archetypes need a fresh `uid` in the `archetype (...)` header line and a filename that matches the inner archetype identifier on line 2.
- Preserve multi-language `translations` blocks and author/organisation metadata when editing.

## Use the openehr-assistant MCP server

This repo is the companion content for that MCP server, and its tools (prefixed `mcp__openehr-assistant__`) are the primary aids for any modelling task here. Follow its documented order of operations:

- **Guide-First**: call `guide_search` / `guide_get` before complex modelling or authoring.
- **Spec-Lookup-First**: consult `guide_get(category="howto", name="spec-lookup")` before fetching anything from `specifications.openehr.org`.
- **Examples-First**: use `examples_search` / `examples_get` for concrete patterns (well-formed OBSERVATION archetypes, FLAT/Structured payloads, AQL).
- Use `terminology_resolve` for terminology codes and `type_specification_get` for per-class RM attribute/invariant detail.
