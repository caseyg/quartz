## 1. Package Scaffold

- [x] 1.1 Fork `quartz-community/plugin-template` into a new repo `quartz-plugin-openwiki` and verify the repo clones, `npm install` succeeds, and `npm run build` produces `dist/index.js`
- [x] 1.2 Update `package.json`: set `name`, `description`, `author`, `repository`, and the `quartz` manifest field (`name: "openwiki"`, `displayName: "OpenWiki"`, `category: "transformer"`, `defaultOptions` with `addTypeTag: true`, `stripBrokenLinkComments: true`) — verify `npm run build` still passes after the change
- [x] 1.3 Delete `src/emitter.ts`, `src/filter.ts`, and the `ExampleComponent` — verify `npm run build` and `npm test` both pass with those files removed and their exports stripped from `src/index.ts`

## 2. Types

- [x] 2.1 Replace `ExampleTransformerOptions` in `src/types.ts` with `OpenWikiTransformerOptions` (fields: `repoBase: string`, `addTypeTag?: boolean`, `stripBrokenLinkComments?: boolean`) — verify TypeScript compiles cleanly with `npm run typecheck`

## 3. textTransform — Broken-link Comment Stripping

- [x] 3.1 Implement `textTransform` in `src/transformer.ts` that strips `<!-- openwiki: broken internal link [...] ... -->` comments via regex when `stripBrokenLinkComments` is `true` — verify by unit test: source containing one such comment returns source with the comment absent; source with no such comment returns unchanged

## 4. remark Plugin — type → tags Promotion

- [x] 4.1 Implement `remarkOpenWikiTags` remark plugin in `src/remark-tags.ts` that reads `vfile.data.frontmatter.type` and appends its string value to `vfile.data.frontmatter.tags` (initialising `tags` to `[]` if absent) — verify by unit test covering: type present with no existing tags, type present with existing tags, no type field (tags untouched), `addTypeTag: false` (tags untouched)
- [x] 4.2 Register `remarkOpenWikiTags` in `markdownPlugins()` in `src/transformer.ts` and verify the plugin appears first in the returned `PluggableList`

## 5. remark Plugin — repo:// Link Rewriting

- [x] 5.1 Implement `remarkOpenWikiLinks` remark plugin in `src/remark-links.ts` that visits MDAST `link` nodes whose `url` starts with `repo://`, decodes the path with `decodeURIComponent`, strips the `repo://` prefix, and prepends the normalised `repoBase` (trailing slash stripped at instantiation) — verify by unit test covering: single `repo://` link rewrites correctly, multiple links all rewrite, non-`repo://` links are unchanged, `repoBase` with trailing slash produces no double slash
- [x] 5.2 Register `remarkOpenWikiLinks` in `markdownPlugins()` in `src/transformer.ts` after `remarkOpenWikiTags` — verify order in the returned `PluggableList`

## 6. Export Wiring

- [x] 6.1 Update `src/index.ts` to export `OpenWikiTransformer` and `OpenWikiTransformerOptions` (remove all `Example*` exports) — verify `npm run build` produces a `dist/index.js` with the correct named exports by running `node --input-type=module -e "import {OpenWikiTransformer} from './dist/index.js'; console.log(typeof OpenWikiTransformer)"`

## 7. Tests

- [x] 7.1 Write `test/transformer.test.ts` covering all spec scenarios: broken-link comment stripping on/off, type→tags promotion with and without existing tags and with the option disabled, `repo://` link rewriting with a simple path, a URL-encoded path, multiple links, and a `repoBase` with a trailing slash — verify `npm test` passes with all cases green
- [x] 7.2 Verify the non-OKF passthrough scenario: a page with no `type`, no `repo://` links, and no broken-link comments is returned byte-for-identical by the transformer — add this as a test case and verify `npm test` passes

## 8. Documentation & Config

- [x] 8.1 Write `README.md` documenting: installation (`npx quartz plugin add <repo-url>`), all three options with types and defaults, the required `quartz.config.yaml` `ignorePatterns` entry (`INSTRUCTIONS.md`), a note on `order: 5` to ensure the transformer runs before other plugins, and a note on the GHE URL accessibility limitation for external users
- [x] 8.2 Verify end-to-end: point a local Quartz build at the Project Synapse openwiki directory with `--directory`, add the plugin to `quartz.config.yaml` with `repoBase: "https://github.ibm.com/productivity-platforms/project-synapse/blob/main"` and `order: 5`, confirm `INSTRUCTIONS.md` is excluded, confirm a page with `type: concept` gains a `concept` tag visible in the built site, and confirm a `repo://` link renders as a clickable GitHub URL
