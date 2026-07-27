# pdf-specialist-plugin

[日本語](./README.ja.md)

An all-in-one plugin for **pdf-specialist**, a subagent for working with PDFs. Installing it gives you the agent definition, the four PDF Family MCP servers, and — through plugin dependencies — the pdf-trust / pdf-publish Skills, in one step.

> Implements pattern 1 (Claude subagent) of the PDF specialist agent design.
> **v0.3.0 has not been through real use yet either** — the details (tool patterns, model,
> delegation triggers) will be corrected against what actually happens in practice.

## Layout

```
pdf-specialist-plugin/
├── .claude-plugin/plugin.json   # Manifest + mcpServers (the four PDF Family servers over npx)
└── agents/pdf-specialist.md     # Subagent definition (delegation triggers + hard rules)
```

The pdf-trust and pdf-publish Skills are **not bundled**. They are declared in
`dependencies` and installed from the same marketplace, so they stay at whatever version their
own repositories publish — see [Skills come from dependencies](#skills-come-from-dependencies).

## Install

```
/plugin marketplace add shuji-bonji/claude-plugins
/plugin install pdf-specialist
```

`pdf-trust` and `pdf-publish` are installed automatically as dependencies and listed at the end of
the install output. Requires **Claude Code v2.1.110 or later** (v2.1.143+ for enable/disable to
propagate to dependencies).

### Environment

| Variable | Server | Required | Purpose |
|---|---|---|---|
| `PDF_SPEC_DIR` | pdf-spec | Yes, to use pdf-spec | Directory holding the specification PDF corpus. Without it pdf-spec fails to start; the other three servers and both Skills still work |
| `PDF_VERIFY_VERAPDF` | pdf-verify | Optional | Path to the veraPDF executable. Falls back to a PATH lookup, then to the built-in rule subset |
| `PDF_VERIFY_TRUST_ANCHORS` | pdf-verify | Optional | Directory of trust anchor certificates. Without it a verdict cannot go beyond `use_with_caution` |
| `PDF_WRITER_FONT` | pdf-writer | In practice, for Japanese output | A single-face `.ttf` / `.otf` (the static Noto Sans JP build is a good choice) |

Set these in your shell environment (launchd, `.zshenv`, and so on). The plugin's `mcpServers` inherit the environment at start-up.

> [!NOTE]
> `${VAR}` expansion inside plugin.json's `env` block **does not work** — the literal string is passed through. This is why the manifest carries no `env` block and relies on inheritance instead.

### Version requirements

- pdf-verify-mcp **v0.7.0+** — `evaluate_policy`, which pdf-trust's four-value verdict depends on.
  **v0.10.0+ recommended** — `verify_integrity` then reports an object-level diff of the revision
  chain, so "what changed after signing" can be answered per object. The verdict itself is unchanged
- pdf-reader-mcp **v0.10.0+** recommended — `locate_objects` (object number → page and rectangle),
  which is what turns that diff into a location
- pdf-writer-mcp **v0.15.0+** recommended — `preserveSignatures`, `tag_form_fields`

The manifest connects with `@latest`, so these are normally satisfied.

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

## Skills come from dependencies

```json
"dependencies": ["pdf-trust", "pdf-publish"]
```

Up to v0.2.0 the two Skills were **copies** kept in `skills/`, re-synced by hand from
[pdf-trust-skill](https://github.com/shuji-bonji/pdf-trust-skill) and
[pdf-publish-skill](https://github.com/shuji-bonji/pdf-publish-skill). A copy that silently went
stale is exactly what happened, so v0.3.0 removed `skills/` and declared the two as dependencies
instead. There is now one source of truth per Skill, and no sync step to forget.

An installed plugin cannot reference files outside its own directory — symlinks and submodules do
not survive installation, because the external files are never copied into the cache. So bundling
and depending were the only two options, and depending is the one that cannot drift.

Both names resolve **within the same marketplace** as this plugin, so no
`allowCrossMarketplaceDependenciesOn` entry is needed. No version range is pinned: each Skill
tracks whatever version its marketplace entry provides. Pinning one would require the marketplace
to carry `pdf-trust--v{version}` git tags, which is a separate convention from the per-repository
release tags in use today.

## Open questions (to be settled by real use)

- [ ] Whether the wildcards in `tools` (`mcp__pdf-reader__*` and friends) match the tool-name prefixes the bundled mcpServers actually produce — some environments use the `mcp__plugin_..._pdf-reader__*` form
- [x] `${VAR}` expansion in plugin.json's `env` **does not work** (confirmed 2026-07-26 against another plugin: `"${PDF_SPEC_DIR}"` arrived verbatim and produced a REGISTRY_ERROR). The `env` block was removed in favour of shell-environment inheritance
- [ ] How reliably delegation fires (wording of the description)
- [ ] Model choice — whether sonnet suffices, or specification-heavy work warrants a stronger model
- [ ] Behaviour when auditing several PDFs at once

## License

MIT
