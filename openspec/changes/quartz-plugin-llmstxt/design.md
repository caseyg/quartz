## Context

This is a new standalone OSS plugin (`quartz-plugin-llmstxt`) modeled on `quartz-community/plugin-template` and `quartz-plugin-openwiki`. It lives outside this Quartz instance — implementation happens in `../quartz-plugin-llmstxt`. The plugin is a pure **emitter**: it receives the full `ProcessedContent[]` array after all filters and transformers have run, and writes static files into the build output directory. No transformer or filter is needed.

Key context from the codebase:
- Quartz emitters implement `{ name, emit(ctx, content, resources), partialEmit(...) }`
- `ProcessedContent = [HtmlRoot, VFile]` — the VFile carries `data.slug`, `data.filePath`, `data.frontmatter`, `data.description`
- `ctx.argv.output` is the build output directory; `ctx.cfg.configuration.baseUrl` is the site's base URL
- The `write` helper pattern from the template: `fs.mkdir(dir, { recursive: true }); fs.writeFile(path, content)`
- `vfile.data.filePath` is the absolute path to the source `.md` file on disk — readable with `fs.readFile`

## Goals / Non-Goals

**Goals:**
- Emit spec-compliant `/llms.txt` with folder-based section grouping
- Emit `/llms-full.txt` concatenation (opt-in, default on)
- Emit `<slug>.md` mirror for every published page (opt-in, default on)
- Zero-config useful defaults; all behavior tunable via options
- Follow `quartz-community/plugin-template` structure exactly (tsup, vitest, changesets, dist committed)
- Published to npm as `quartz-plugin-llmstxt` under `caseyg` GitHub account

**Non-Goals:**
- No `<link rel="alternate">` transformer — no HTML injection
- No filter plugin
- No UI components
- No tag-based grouping — folder hierarchy only
- No per-page description scraping beyond `vfile.data.description` (already computed by `@quartz-community/description` if present)

## Decisions

### D1: Emitter-only, no transformer

**Decision:** Ship only a `LlmsTxtEmitter`. No rehype transformer to inject `<link rel="alternate">` headers.

**Rationale:** The link injection requires touching every page's HAST tree during build, adding complexity and a performance cost. The spec recommends it but doesn't require it. Agents that start from `llms.txt` don't need it. Can be added in a future version.

**Alternative considered:** Ship `LlmsTxtTransformer` that injects `rel="alternate"` and `rel="describedby"` into `<head>`. Rejected: out of scope for v1, increases surface area.

---

### D2: Source `.md` mirrors from `vfile.data.filePath`, not HAST serialization

**Decision:** Read the raw source `.md` file from disk using `vfile.data.filePath` for each mirror.

**Rationale:** The raw source is what the author wrote — clean markdown, possibly with wikilinks and frontmatter, which LLMs handle well. Serializing HAST back to markdown is lossy (loses frontmatter, produces verbose output) and requires `hast-util-to-markdown` as an additional dependency. Reading the source file is a single `fs.readFile` call with zero dependencies.

**Alternative considered:** Serialize the processed HAST tree. Rejected: lossy, heavier, would strip frontmatter.

---

### D3: Folder prefix derived from slug, not frontmatter

**Decision:** Section grouping is determined by the first path segment of the slug (e.g. `notes/page` → `notes`). Root-level pages (no `/` in slug) go to a `Pages` section. Custom display names via `sections` option.

**Rationale:** Quartz slugs directly reflect the content folder structure. No frontmatter convention to establish, works out of the box for any Quartz site. Mirrors how users already organize their vaults.

**Alternative considered:** Group by frontmatter `category` or `type` field. Rejected: requires user-side conventions, creates setup friction.

---

### D4: Links in `llms.txt` point to `slug.md`, not `slug` or `slug.html`

**Decision:** All links in `llms.txt` are `https://<baseUrl>/<slug>.md`.

**Rationale:** The spec recommends linking to the LLM-friendly markdown version of pages. Agents following links from `llms.txt` get clean markdown directly rather than HTML they'd need to parse. Consistent with the `.md` mirrors the plugin emits.

---

### D5: `partialEmit` re-emits everything

**Decision:** `partialEmit` calls the same `emitAll` function as `emit` — full re-emit every time.

**Rationale:** `llms.txt` and `llms-full.txt` are derived from the entire content list. There's no correct incremental strategy — any page change could affect section ordering or full-text content. The files are small enough that a full re-emit is fast.

---

### D6: Plugin structure — single export file, no components

**Decision:** `src/index.ts` exports only `LlmsTxtEmitter` and `LlmsTxtEmitterOptions`. No transformer, filter, or component exports.

**Rationale:** Keeps the plugin surface minimal and the `package.json` `quartz.category` as `"emitter"`. Future additions (transformer for `<link>` injection) can be added as additional named exports without breaking changes.

---

### D7: `llms-full.txt` separator format

**Decision:** Between pages, use:
```
---

## {title} ({slug})

{source markdown}
```

**Rationale:** The `---` separator is conventional in markdown for section breaks. The `## {title} ({slug})` heading lets an agent scan for specific content. Readable both by humans and LLMs.

## Risks / Trade-offs

- **[Risk] Large sites produce large `llms-full.txt`** → Mitigation: `emitFullTxt: false` opt-out. The file is only read on demand by agents; it doesn't affect page load.
- **[Risk] `vfile.data.filePath` may be undefined for virtual/generated pages** (e.g. folder index pages generated by Quartz) → Mitigation: guard with `if (!filePath) continue` — skip mirror emit for pages with no source file. Document in README.
- **[Risk] Source `.md` files contain wikilinks and OFM syntax** that differ from rendered HTML → This is acceptable — LLMs handle `[[wikilinks]]` well and the raw source is more faithful than an HTML parse.
- **[Trade-off] No `<link rel="alternate">` injection** means agents browsing individual pages won't auto-discover the markdown version → Accepted for v1; documented in README as a known limitation.

## Open Questions

None — all decisions are resolved. Implementation can proceed directly from this design and the specs.
