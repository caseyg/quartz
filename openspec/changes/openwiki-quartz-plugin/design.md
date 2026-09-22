## Context

See `proposal.md — Why` for motivation.

The Quartz plugin ecosystem uses a factory-function pattern: a plugin is a function that returns an object with a `name` and optional lifecycle hooks (`textTransform`, `markdownPlugins`, `htmlPlugins`). The `@quartz-community/plugin-template` provides the canonical package scaffold (tsup build, vitest tests, `@quartz-community/types` peer dependency, `quartz` manifest field in `package.json`).

OpenWiki's three conventions that need handling:

1. **Broken-link comments** — raw HTML comments injected by OpenWiki into the markdown source; must be stripped before the unified pipeline parses the document
2. **`type` frontmatter field** — OKF page type (concept, workflow, overview, etc.); not a standard Quartz field, so it needs to be mapped to `tags` for Explorer/tag-page to pick it up
3. **`repo://` links** — URL-encoded paths referencing source files in the original repository; appear as standard markdown links in the body and need their href scheme rewritten

## Goals / Non-Goals

**Goals:**
- Implement all three transforms as a single `QuartzTransformerPlugin` following the plugin-template structure
- Package is independently installable and reusable across any Quartz+OpenWiki deployment
- Zero changes to Quartz core

**Non-Goals:**
- Filter plugin for the OKF root manifest `index.md` (handled by `ignorePatterns` or `draft: true` in the openwiki repo) — v2
- Claims footnotes from `.claims/` sidecar JSON — v2
- `OpenWikiSourceBadge` UI component — v2
- Support for `repo://` references inside frontmatter `sources[]` (these are metadata, not navigable links)

## Decisions

### Decision 1: Three transforms in one plugin, not three plugins

**Chosen:** Single `OpenWikiTransformer` plugin exporting one factory function.

**Rationale:** All three transforms are tightly scoped to OKF content. Splitting them into separate plugins adds config overhead for users (three entries in `quartz.config.yaml` instead of one) with no benefit — they always need to run together on the same content, and none has standalone utility outside the OKF context.

**Alternative considered:** Separate `OpenWikiTagPlugin`, `OpenWikiLinksPlugin`, `OpenWikiBrokenLinksPlugin`. Rejected: unnecessary user-facing complexity.

---

### Decision 2: `textTransform` for comment stripping, remark plugins for the rest

**Chosen:** Strip broken-link comments in `textTransform` (raw string, before unified parses the document). Promote `type → tags` and rewrite `repo://` links as remark plugins in `markdownPlugins`.

**Rationale:**
- `textTransform` runs before the unified pipeline. A regex replace on the raw string is the simplest and most correct approach for stripping HTML comments — remark's HTML node visitor would work too, but would require matching on HTML node content after parsing, which is more fragile.
- The `type → tags` promotion must happen at the MDAST stage so the value is in `vfile.data.frontmatter.tags` before downstream plugins (e.g. `crawl-links`, `description`) read it. A remark plugin that modifies `vfile.data.frontmatter` is the natural hook.
- `repo://` link rewriting walks MDAST `link` nodes — the remark stage is exactly right. The decoded path must be used (not the raw URL-encoded form) so the GitHub URL renders as a clean human-readable string.

**Alternative considered:** `htmlPlugins` (rehype) for `repo://` rewriting. Rejected: link hrefs at the HAST stage are harder to target cleanly and the fix needs to happen before `crawl-links` processes links.

---

### Decision 3: `repoBase` trailing-slash normalisation inside the plugin

**Chosen:** The plugin normalises `repoBase` at instantiation time by stripping any trailing slash, then joins with the decoded path using `/`.

**Rationale:** User-facing config should be forgiving. Requiring an exact format in `repoBase` is an unnecessary footgun — both `https://github.ibm.com/org/repo/blob/main` and `https://github.ibm.com/org/repo/blob/main/` should produce identical output.

---

### Decision 4: Fork `quartz-community/plugin-template` as the package scaffold

**Chosen:** Create the plugin package by forking `quartz-community/plugin-template`.

**Rationale:** The template provides the correct tsup build config, vitest test setup, `@quartz-community/types` dependency, ESLint/Prettier config, and `quartz` manifest field in `package.json`. Building from scratch would replicate all of this. The template's `SINGLETON_EXTERNALS` (preact, vfile, unified) is important for correctness — it ensures the plugin shares the same unified instance as Quartz core rather than bundling its own.

---

### Decision 5: `category: "transformer"` in `package.json` quartz manifest

**Chosen:** Single `"transformer"` category in the manifest, even though a filter was discussed.

**Rationale:** The three content transforms are all transformer concerns. The filtering needs (excluding `INSTRUCTIONS.md`, handling the OKF root `index.md`) are handled by `ignorePatterns` in `quartz.config.yaml` and/or a `draft: true` frontmatter field on the root manifest — both are zero-code solutions available today. Adding a filter category to this package for v1 would add test surface, loader complexity, and a second config entry in `quartz.config.yaml` without meaningfully improving the user experience. The filter can be added as a v2 named export (`OpenWikiFilter`) without any breaking change to users who only use the transformer.

## Risks / Trade-offs

**`vfile.data.frontmatter` mutation order** → Downstream plugins that run before `OpenWikiTransformer` will not see the promoted tag. Mitigation: set `order: 5` in `quartz.config.yaml` (lower than all other transformers) to ensure it runs first. Document this in the plugin README.

**`repo://` links resolve to GHE URLs inaccessible to external users** → If the Quartz site is ever published outside the IBM network, source links will 404 for unauthenticated users. Mitigation: add a `showSourceLinks: false` option in v2 that renders `repo://` references as plain text rather than links. For v1 this is acceptable for an internal audience.

**No `.claims/` filtering in v1** → The `.claims/` directory is excluded automatically by Quartz's glob behaviour (`dot: false` default in globby). `INSTRUCTIONS.md` must be added to `ignorePatterns` manually. The OKF root `index.md` (containing `okf_version`) is either left to render (it's a valid link list) or excluded with `draft: true`. Mitigation: document the required `ignorePatterns` configuration clearly in the README.

**Remark plugin order within `markdownPlugins()`** → The type→tags plugin must run before any plugin that reads `frontmatter.tags`. The two remark plugins exported by this transformer are order-independent with respect to each other. Mitigation: register type→tags before links in the returned `PluggableList`.

## Open Questions

- Should `repoBase` default to a well-known value (e.g. the IBM GHE org/repo) or remain a required field with no default? Leaning toward required with no default — a missing `repoBase` should be an obvious config error, not a silent misconfiguration pointing at the wrong repo. Can decide at implementation time without changing the spec or task breakdown.
