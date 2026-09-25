## Purpose

Enables Quartz sites to emit a spec-compliant `/llms.txt` index, an optional `/llms-full.txt` concatenation, and clean `.md` mirrors of every published page so that LLM agents can discover and consume site content without parsing HTML.

## ADDED Requirements

### Requirement: Emit llms.txt index

The plugin SHALL emit a `/llms.txt` file at the site root on every build. The file SHALL conform to the llms.txt v2 specification: an H1 with the site title, a blockquote with a short description, and zero or more H2 sections each containing a markdown list of links. Every link SHALL point to the `.md` mirror URL of the target page (not the `.html` URL). Pages SHALL be grouped into H2 sections by their top-level folder prefix. A page at slug `notes/my-page` belongs to the `Notes` section; a page at the root (no folder prefix) belongs to a `Pages` section. Sections SHALL appear in lexicographic order by section name, with `Optional` always last when present. The `## Optional` section SHALL contain links for any slug that matches a configured `optionalPrefixes` entry. Slugs listed in `excludeFromIndex` SHALL NOT appear in `llms.txt`.

#### Scenario: Default build with folder-structured content

- **WHEN** the site has pages at slugs `notes/page-a`, `notes/page-b`, and `projects/thing`
- **THEN** `llms.txt` contains `## Notes` with links to `notes/page-a.md` and `notes/page-b.md`, and `## Projects` with a link to `projects/thing.md`

#### Scenario: Root-level pages

- **WHEN** a page has slug `about` (no folder prefix)
- **THEN** `llms.txt` places it under a `## Pages` section

#### Scenario: Optional section

- **WHEN** `optionalPrefixes` includes `"archive"` and a page has slug `archive/old-note`
- **THEN** the link appears under `## Optional` instead of `## Archive`

#### Scenario: Excluded slugs

- **WHEN** `excludeFromIndex` includes `"private/secret"`
- **THEN** `llms.txt` contains no link to that slug, but `private/secret.md` is still emitted

#### Scenario: Custom section names

- **WHEN** `sections` maps `"notes"` to `"My Notes"`
- **THEN** the section heading in `llms.txt` is `## My Notes`

#### Scenario: Description override

- **WHEN** `description` option is set
- **THEN** the blockquote in `llms.txt` uses that string instead of the site title

### Requirement: Emit per-page markdown mirrors

The plugin SHALL emit a `<slug>.md` file for every page in the published content set when `emitMarkdownMirrors` is `true` (the default). Each `.md` file SHALL contain the raw source markdown read from the page's original file on disk (`vfile.data.filePath`). The output path SHALL be the page's slug with a `.md` extension appended (e.g. slug `notes/my-page` → `notes/my-page.md`). Pages excluded via `excludeFromIndex` SHALL still receive `.md` mirrors.

#### Scenario: Mirror emitted for each page

- **WHEN** a page with slug `notes/my-page` is published and `emitMarkdownMirrors` is `true`
- **THEN** a file `notes/my-page.md` is written to the output directory containing the page's source markdown

#### Scenario: Mirror disabled

- **WHEN** `emitMarkdownMirrors` is `false`
- **THEN** no `.md` mirror files are emitted

#### Scenario: Index page mirror

- **WHEN** a page has slug `notes` (folder index, source is `notes/index.md`)
- **THEN** the mirror is written to `notes.md`

### Requirement: Emit llms-full.txt concatenation

The plugin SHALL emit a `/llms-full.txt` file when `emitFullTxt` is `true` (the default). The file SHALL contain the raw source markdown of every published page concatenated together, separated by a `---` horizontal rule and a heading identifying the page title and slug. The order SHALL match the order pages appear in `llms.txt` (grouped by section, lexicographic within each section).

#### Scenario: Full text emitted by default

- **WHEN** `emitFullTxt` is not set (defaults to `true`) and the site has three pages
- **THEN** `llms-full.txt` is present in the output and contains the source markdown of all three pages

#### Scenario: Full text disabled

- **WHEN** `emitFullTxt` is `false`
- **THEN** no `llms-full.txt` file is emitted

#### Scenario: Excluded pages absent from full text

- **WHEN** a slug is listed in `excludeFromIndex`
- **THEN** its content does NOT appear in `llms-full.txt`

### Requirement: Plugin options

The plugin SHALL accept a configuration object with the following optional fields. All fields SHALL have defaults such that zero configuration produces a useful output.

| Option | Type | Default | Description |
|---|---|---|---|
| `emitMarkdownMirrors` | `boolean` | `true` | Emit `.md` mirror for every page |
| `emitFullTxt` | `boolean` | `true` | Emit `/llms-full.txt` |
| `sections` | `Record<string, string>` | `{}` | Folder prefix → display name overrides |
| `optionalPrefixes` | `string[]` | `[]` | Slug prefixes routed to `## Optional` |
| `excludeFromIndex` | `string[]` | `[]` | Slug prefixes excluded from `llms.txt` and `llms-full.txt` |
| `description` | `string` | `""` | Override for the site description blockquote |

#### Scenario: Zero-config usage

- **WHEN** the plugin is registered with no options
- **THEN** `llms.txt`, `llms-full.txt`, and `.md` mirrors are all emitted with auto-derived section names

### Requirement: Incremental rebuild parity

The plugin SHALL produce identical output whether triggered by a full build (`emit`) or an incremental rebuild (`partialEmit`). Because generating `llms.txt` and `llms-full.txt` requires the full content list, `partialEmit` SHALL re-emit all files unconditionally.

#### Scenario: Partial emit produces same files as full emit

- **WHEN** a single page changes and `partialEmit` is called with the full content list
- **THEN** all output files match what a full `emit` would produce
