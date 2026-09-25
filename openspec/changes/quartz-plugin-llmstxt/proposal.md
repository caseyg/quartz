## Why

LLMs and AI agents increasingly consume documentation and knowledge bases directly. Quartz sites have no machine-friendly entry point today — agents must parse HTML, lose structure, and waste context. The [llms.txt specification](https://llmstxt.org) defines a lightweight standard for exactly this: a curated `/llms.txt` index plus clean `.md` mirrors of every page. Publishing these from a Quartz site makes the entire knowledge base immediately usable by any AI agent.

## What Changes

- New standalone OSS plugin `quartz-plugin-llmstxt` published to `github.com/caseyg/quartz-plugin-llmstxt` and npm as `quartz-plugin-llmstxt`, following the `quartz-community/plugin-template` structure.
- Emits `/llms.txt` — a spec-compliant markdown index with site title, description blockquote, and pages grouped by folder hierarchy into H2 sections, with links pointing to `.md` mirror URLs.
- Emits `/llms-full.txt` — concatenation of all page markdown content for agents that want everything in one file (opt-in, on by default, disable with `emitFullTxt: false`).
- Emits `<slug>.md` mirrors for every published page — raw source markdown read from each file's original path on disk, written at the corresponding output slug.
- Folder hierarchy is auto-derived into section names (e.g. `notes/` → `## Notes`); custom section name overrides available via `sections` option.
- Configurable `optionalPrefixes` puts matching slugs into a conventional `## Optional` section in `llms.txt`.
- Configurable `excludeFromIndex` keeps slugs out of `llms.txt` while still emitting their `.md` mirrors.
- No transformer, no filter, no UI components — emitter only.

## Capabilities

### New Capabilities

- `llmstxt-emission`: Emits `/llms.txt`, `/llms-full.txt`, and per-page `.md` mirror files during the Quartz build, following the llms.txt v2 specification.

### Modified Capabilities

<!-- none — this is a new standalone plugin, no existing quartz capabilities change -->

## Impact

- **New repo**: `../quartz-plugin-llmstxt` (sibling to `../quartz-plugin-openwiki`)
- **Modeled after**: `quartz-community/plugin-template` (emitter pattern) and `@quartz-community/content-index` (emitter that generates auxiliary files from full content list)
- **Dependencies**: `@quartz-community/types` (for `QuartzEmitterPlugin`, `ProcessedContent`, `BuildCtx`, `FilePath`, `FullSlug`); `node:fs/promises` and `node:path` (built-in)
- **No changes** to this Quartz instance or any existing plugins
- **Quartz config**: users add the plugin via `source: "github:caseyg/quartz-plugin-llmstxt"` in `quartz.config.yaml`
