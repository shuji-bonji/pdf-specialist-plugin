# pdf-specialist-plugin

[日本語](./README.ja.md)

An all-in-one plugin for **pdf-specialist**, a subagent for working with PDFs. Installing it gives you the agent definition and — through plugin dependencies — the four PDF Family MCP servers and the pdf-trust / pdf-publish Skills, in one step.

> Implements pattern 1 (Claude subagent) of the PDF specialist agent design.
> **v0.4.0 has not been through real use yet either** — the remaining details (model, delegation
> triggers) will be corrected against what actually happens in practice.

## Layout

```
pdf-specialist-plugin/
├── .claude-plugin/plugin.json   # Manifest + dependencies
└── agents/pdf-specialist.md     # Subagent definition (delegation triggers + hard rules)
```

Nothing else is bundled. The four MCP servers and the two Skills are all
[declared as dependencies](#everything-comes-from-dependencies) and installed from the same
marketplace.

## Install

```
/plugin marketplace add shuji-bonji/claude-plugins
/plugin install pdf-specialist
```

The six dependencies — `pdf-trust`, `pdf-publish` and the four MCP plugins — are installed
automatically and listed at the end of the install output. Requires **Claude Code v2.1.110 or
later** (v2.1.143+ for enable/disable to propagate to dependencies).

### Environment

| Variable | Server | Required | Purpose |
|---|---|---|---|
| `PDF_SPEC_DIR` | pdf-spec | Yes, to use pdf-spec | Directory holding the specification PDF corpus. Without it pdf-spec fails to start; the other three servers and both Skills still work |
| `PDF_VERIFY_VERAPDF` | pdf-verify | Optional | Path to the veraPDF executable. Falls back to a PATH lookup, then to the built-in rule subset |
| `PDF_VERIFY_TRUST_ANCHORS` | pdf-verify | Optional | Directory of trust anchor certificates. Without it a verdict cannot go beyond `use_with_caution` |
| `PDF_WRITER_FONT` | pdf-writer | In practice, for Japanese output | A single-face `.ttf` / `.otf` (the static Noto Sans JP build is a good choice) |

Set these in your shell environment (launchd, `.zshenv`, and so on). The servers come from the four
MCP plugins and inherit that environment at start-up.

> [!NOTE]
> `${VAR}` expansion inside a plugin.json `env` block **does not work** — the literal string is passed through. Every manifest in this family relies on shell-environment inheritance instead.

### Version requirements

- pdf-verify-mcp **v0.7.0+** — `evaluate_policy`, which pdf-trust's four-value verdict depends on.
  **v0.10.0+ recommended** — `verify_integrity` then reports an object-level diff of the revision
  chain, so "what changed after signing" can be answered per object. The verdict itself is unchanged.
  **v0.17.0+ recommended for audit reports** — whether the revision list is the whole history
  (`revisionChain`, v0.16.0) and whether the `startxref` count and the listed revisions differ for a
  stated reason (`revisionCountAgreement`, v0.17.0) are then fields; pdf-trust v0.7.0 reads them
  instead of matching prose in `notes`
- pdf-reader-mcp **v0.10.0+** recommended — `locate_objects` (object number → page and rectangle),
  which is what turns that diff into a location. **v0.11.0+** adds the other direction:
  `extract_structured_text` with `include_bbox` locates a *structure element*, so "annotate this
  paragraph" needs no coordinate from the user either
- pdf-writer-mcp **v0.15.0+** recommended — `preserveSignatures`, `tag_form_fields`.
  **v0.16.0+** adds PDF 2.0 output and the PDF/A-4 / PDF/A-4f containers (`ensure_pdfa` `flavour`)

Each MCP plugin connects with `@latest`, so these are normally satisfied. No version range is
pinned in `dependencies` (see below).

## Usage

Ask the main agent as you normally would; the description's triggers route the work to pdf-specialist:

- "Can I trust this contract PDF?" → pdf-trust (`profile=contract`)
- "Turn this Markdown into a tagged, PDF/UA-conformant PDF" → pdf-publish
- "Is a document title mandatory for tagged PDF? Which clause says so?" → pdf-spec lookup

## Design rules

1. **Never override a verdict** — `evaluate_policy` and veraPDF decide; the agent only explains
2. **Keep declaration and conformance apart** — after any `ensure_*`, run `validate_conformance` with the matching flavour
3. **Absolute paths, always `outputPath`** — otherwise a few MB of PDF comes back as millions of characters of base64
4. **Say what was skipped** — anything an unconnected MCP would have covered is reported as not checked, not as fine
5. **Answer in two layers** — a machine-readable summary, then a report for a human

## Everything comes from dependencies

```json
"dependencies": [
  "pdf-trust", "pdf-publish",
  "pdf-reader-mcp", "pdf-spec-mcp", "pdf-verify-mcp", "pdf-writer-mcp"
]
```

This plugin ships an agent definition and nothing else. It got there in two steps, both driven by
the same failure mode.

**v0.3.0 — the Skills.** Up to v0.2.0 they were **copies** in `skills/`, re-synced by hand from
[pdf-trust-skill](https://github.com/shuji-bonji/pdf-trust-skill) and
[pdf-publish-skill](https://github.com/shuji-bonji/pdf-publish-skill). A copy silently went stale,
which is what copies do. An installed plugin cannot reference files outside its own directory —
symlinks and submodules do not survive installation, because the external files are never copied
into the cache. So bundling and depending were the only two options, and depending is the one that
cannot drift.

**v0.4.0 — the MCP servers.** The manifest used to define all four inline, which collided with the
standalone MCP plugins: Claude Code skipped this plugin's copies as duplicates of the same command.
Worse, an MCP server's tools are named `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`, so
the prefix depended on **which plugin won the duplicate check** — that is, on what else the user
had installed. A static `tools` allowlist in the agent cannot be correct for both cases. Declaring
the MCP plugins as dependencies makes the names deterministic.

That mattered more than it sounds: through v0.3.0 the agent's allowlist read `mcp__pdf-reader__*`,
which matches nothing under the documented naming, so **the agent had no PDF tools at all**. The
patterns are now spelled out in full in `agents/pdf-specialist.md`.

All six names resolve **within the same marketplace** as this plugin, so no
`allowCrossMarketplaceDependenciesOn` entry is needed. No version range is pinned: each dependency
tracks whatever version its marketplace entry provides. Pinning one would require the marketplace
to carry `pdf-trust--v{version}` git tags, which is a separate convention from the per-repository
release tags in use today.

## Open questions (to be settled by real use)

- [x] **Dependency resolution works** (verified 2026-07-27 on Claude Code v2.1.212): a clean
      `claude plugin install pdf-specialist@shuji-bonji` reported `(+ 2 dependencies: pdf-publish,
      pdf-trust)`; uninstalling reported pdf-trust as an auto-installed dependency eligible for
      `claude plugin prune`; and disabling pdf-trust while pdf-specialist is enabled is refused
      with a chained command. What is *not* yet verified is delegation and day-to-day use
- [x] **The `tools` wildcards did not match** (found 2026-07-27). The documented naming is `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`, so `mcp__pdf-reader__*` matched nothing and the allowlist left the agent with only Read / Glob / Skill. Fixed in v0.4.0 by writing the full prefixes and moving the MCP servers to dependencies so the names stop depending on what else is installed
- [x] `${VAR}` expansion in plugin.json's `env` **does not work** (confirmed 2026-07-26 against another plugin: `"${PDF_SPEC_DIR}"` arrived verbatim and produced a REGISTRY_ERROR). The `env` block was removed in favour of shell-environment inheritance
- [ ] How reliably delegation fires (wording of the description)
- [ ] Model choice — whether sonnet suffices, or specification-heavy work warrants a stronger model
- [ ] Behaviour when auditing several PDFs at once

## License

MIT
