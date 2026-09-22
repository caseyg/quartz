## Why

OpenWiki generates structured markdown wikis from document repositories, but its built-in visualizer is a limited static site generator. Quartz offers far richer navigation, search, graph views, and extensibility. A Quartz transformer plugin is needed to bridge the two: handling OpenWiki's OKF frontmatter format, rewriting internal `repo://` source links to real GitHub URLs, surfacing the `type` field as Quartz tags for navigation, and stripping build-time comment annotations that should never reach readers.

## What Changes

- New standalone community plugin package `quartz-plugin-openwiki` (forked from `quartz-community/plugin-template`)
- Implements a `QuartzTransformerPlugin` with three OKF-specific transforms:
  - **`textTransform`**: strips `<!-- openwiki: broken internal link [...] -->` HTML comments before parsing
  - **remark plugin**: promotes `frontmatter.type` (e.g. `concept`, `workflow`, `overview`) into `frontmatter.tags` so Quartz tag pages and explorer navigation work
  - **remark plugin**: rewrites `repo://url-encoded-path` link hrefs in the markdown body to fully-qualified GitHub URLs using a configurable `repoBase`
- Plugin is configurable via `repoBase` (required), `addTypeTag` (default: `true`), and `stripBrokenLinkComments` (default: `true`)
- Quartz sites using an OpenWiki repo as `argv.directory` also configure `ignorePatterns: ["INSTRUCTIONS.md"]` to exclude OpenWiki's internal config file (dot-prefixed files like `.claims/` are excluded automatically by Quartz's glob behavior)

## Capabilities

### New Capabilities

- `openwiki-transformer`: A Quartz transformer plugin that processes OpenWiki Knowledge Format (OKF) frontmatter and markdown conventions, making OpenWiki-generated content repos renderable as first-class Quartz sites

### Modified Capabilities

<!-- No existing capabilities are changing — this is a new standalone plugin package -->

## Impact

- **New package**: `quartz-plugin-openwiki` — standalone npm/git package, not a change to the Quartz core
- **Quartz core**: no changes; plugin uses only the public `QuartzTransformerPlugin` API
- **Dependencies**: `unist-util-visit` (remark/rehype AST traversal, already available in quartz-community ecosystem); no new native dependencies
- **Consumers**: Any Quartz site pointing `argv.directory` at an OpenWiki repo adds the plugin to `quartz.config.yaml` with a `repoBase` option pointing at their source GitHub repo
- **Out of scope (v2)**: filter plugin for the root OKF manifest `index.md`; claims footnotes from `.claims/` sidecar JSON files; `OpenWikiSourceBadge` component
