## 1. Repo Setup

- [x] 1.1 Use `quartz-community/plugin-template` as the template to create `../quartz-plugin-llmstxt` — clone or degit the template into that directory and verify all template files are present (`src/`, `dist/`, `test/`, `types/`, `package.json`, `tsup.config.ts`, `vitest.config.ts`, `tsconfig.json`, `tsconfig.build.json`, `.eslintrc.json`, `.prettierrc`, `.changeset/`, `.github/workflows/`, `ARCHITECTURE.md`, `AGENTS.md`, `EXAMPLES.md`, `CHANGELOG.md`, `LICENSE`, `README.md`)
- [x] 1.2 Update `package.json`: set `name` to `quartz-plugin-llmstxt`, `author` to `Casey Gollan`, `homepage` and `repository.url` to `https://github.com/caseyg/quartz-plugin-llmstxt`, keywords to `["quartz", "quartz-plugin", "llmstxt", "llms.txt", "ai", "agents"]`, and the `quartz` manifest block (`name: "llmstxt"`, `displayName: "LLMs.txt"`, `category: "emitter"`, `defaultEnabled: true`, `defaultOptions` matching all option defaults, `optionSchema` covering all options) — verify `npm install` succeeds
- [x] 1.3 Remove all template example source files (`src/transformer.ts`, `src/filter.ts`, `src/emitter.ts`, `src/components/`, `src/i18n/`, `src/util/`) and their test counterparts, leaving only `src/index.ts` and `src/types.ts` as stubs — verify `npm run build` reports no missing file errors (will fail on import errors until step 2)

## 2. Types

- [x] 2.1 Write `src/types.ts` defining the exported `LlmsTxtEmitterOptions` interface with all six fields (`emitMarkdownMirrors`, `emitFullTxt`, `sections`, `optionalPrefixes`, `excludeFromIndex`, `description`) with JSDoc comments and correct TypeScript types — verify `npm run typecheck` passes on this file alone

## 3. Core Emitter Implementation

- [x] 3.1 Write the `sectionForSlug(slug, options)` helper in `src/emitter.ts`: given a slug string and options, return the display-name string for the section it belongs to — applies `optionalPrefixes` check first, then `sections` override, then auto-derives from first path segment (capitalizing it), then falls back to `"Pages"` for root slugs — verify with inline unit test in `test/emitter.test.ts` covering: root slug, folder slug, custom section name, optional prefix
- [x] 3.2 Write the `generateLlmsTxt(cfg, content, options)` function: groups `ProcessedContent[]` by section using `sectionForSlug`, sorts sections lexicographically with `Optional` last, builds the markdown string per spec (H1 from `cfg.configuration.pageTitle`, blockquote from `options.description || cfg.configuration.pageTitle`, H2 sections each with a list of `[title](https://<baseUrl>/<slug>.md): description` links), excludes slugs in `excludeFromIndex` — verify with unit test covering: section grouping, Optional ordering, exclude list, description override, link format
- [x] 3.3 Write the `generateLlmsFullTxt(content, options)` function: filters out `excludeFromIndex` slugs, reads source file for each remaining page from `vfile.data.filePath`, concatenates with `---\n\n## {title} ({slug})\n\n{source}` separators — verify with unit test using mocked `fs.readFile` that confirms separator format and exclusion behavior
- [x] 3.4 Write the `LlmsTxtEmitter` factory function in `src/emitter.ts`: merge user options with defaults, implement `emit(ctx, content, resources)` that calls `generateLlmsTxt`, optionally `generateLlmsFullTxt`, optionally per-page mirror writes, and returns all emitted `FilePath[]`; implement `partialEmit` that delegates to the same `emitAll` function — verify `npm run typecheck` passes and the emitter satisfies the `QuartzEmitterPlugin` type
- [x] 3.5 Update `src/index.ts` to export `LlmsTxtEmitter` and `LlmsTxtEmitterOptions` (and re-export relevant types from `@quartz-community/types`) — verify `npm run build` succeeds and `dist/index.js` + `dist/index.d.ts` are present

## 4. Tests

- [x] 4.1 Write integration-style tests in `test/emitter.test.ts` covering the five spec scenarios: (a) folder-structured content produces correct sections, (b) root-level pages go to `## Pages`, (c) `optionalPrefixes` routes to `## Optional`, (d) `excludeFromIndex` omits from index but still emits mirror, (e) zero-config defaults produce all three output types — verify `npm test` passes with no skipped tests
- [x] 4.2 Add an edge-case test: page with `vfile.data.filePath === undefined` (virtual page) is skipped for mirror emit without throwing — verify test passes

## 5. Documentation

- [x] 5.1 Write `README.md`: installation via `npx quartz plugin add github:caseyg/quartz-plugin-llmstxt`, `quartz.config.yaml` registration snippet, full options table matching the spec, output file descriptions (`/llms.txt`, `/llms-full.txt`, `<slug>.md`), link to llmstxt.org spec, known limitation (no `<link rel="alternate">` injection) — verify README renders correctly on GitHub (check markdown syntax manually)
- [x] 5.2 Write `EXAMPLES.md` with two examples: (a) default zero-config usage, (b) customized usage with `sections`, `optionalPrefixes`, and `excludeFromIndex` set — verify file is present and syntactically valid markdown

## 6. Build Verification & Release Prep

- [x] 6.1 Run `npm run check` (typecheck + lint + format + test) and confirm it passes with zero warnings — fix any lint or format issues until the command exits cleanly
- [x] 6.2 Commit `dist/` to the repo (template convention: pre-built dist ships in the repo) — verify `git status` shows `dist/` is tracked and up to date after build
- [x] 6.3 Create initial changeset with `npx changeset` (minor bump, description: "Initial release — LlmsTxtEmitter with llms.txt, llms-full.txt, and .md mirror emission") — verify `.changeset/` contains the new changeset file
- [x] 6.4 Push repo to `github.com/caseyg/quartz-plugin-llmstxt` and verify GitHub Actions CI (`npm run check` workflow) passes on the first push
