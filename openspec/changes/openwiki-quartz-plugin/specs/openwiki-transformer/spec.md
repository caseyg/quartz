## Purpose

A Quartz transformer plugin that processes OpenWiki Knowledge Format (OKF) frontmatter and markdown conventions, enabling OpenWiki-generated content repositories to be rendered as first-class Quartz sites without manual content editing.

## ADDED Requirements

### Requirement: Plugin accepts configuration options

The plugin SHALL accept an options object at instantiation time with the following fields:

- `repoBase` (string, required): the base URL used to resolve `repo://` links (e.g. `https://github.ibm.com/org/repo/blob/main`)
- `addTypeTag` (boolean, optional, default `true`): when true, the OKF `type` frontmatter field is appended to the page's `tags` array
- `stripBrokenLinkComments` (boolean, optional, default `true`): when true, OpenWiki broken-link HTML comments are stripped from the raw markdown before parsing

#### Scenario: Plugin instantiated with only repoBase

- **WHEN** the plugin is instantiated with `{ repoBase: "https://github.ibm.com/org/repo/blob/main" }`
- **THEN** `addTypeTag` defaults to `true` and `stripBrokenLinkComments` defaults to `true`

#### Scenario: Plugin instantiated with all options explicitly set

- **WHEN** the plugin is instantiated with all three options provided
- **THEN** the provided values override all defaults

---

### Requirement: Broken internal link comments are stripped

The plugin SHALL remove OpenWiki broken-link HTML comments from the raw markdown source before any parsing occurs.

OpenWiki annotates unresolvable internal links with comments of the form:
```
<!-- openwiki: broken internal link [../path/to/missing.md] ... -->
```

These comments SHALL be removed from the source string so they do not appear in rendered output.

#### Scenario: Page contains a broken-link comment

- **WHEN** `stripBrokenLinkComments` is `true` and the raw markdown contains one or more `<!-- openwiki: broken internal link ... -->` comments
- **THEN** those comments are absent from the processed output HTML

#### Scenario: stripBrokenLinkComments disabled

- **WHEN** `stripBrokenLinkComments` is `false`
- **THEN** broken-link comments are left in the source and will appear as HTML comments in the rendered output

#### Scenario: Page contains no broken-link comments

- **WHEN** the raw markdown contains no OpenWiki broken-link comments
- **THEN** the source is returned unchanged by the strip step

---

### Requirement: OKF type field is promoted to Quartz tags

The plugin SHALL read the `type` field from a page's OKF frontmatter and append its value to the page's `tags` array.

This enables Quartz tag pages and the Explorer component to organise OpenWiki content by its OKF type (e.g. `concept`, `workflow`, `overview`, `operation`).

#### Scenario: Page has a type field and no existing tags

- **WHEN** `addTypeTag` is `true` and the page frontmatter has `type: concept` and no `tags` field
- **THEN** the page's `tags` array is `["concept"]` after transformation

#### Scenario: Page has a type field and existing tags

- **WHEN** `addTypeTag` is `true` and the page frontmatter has `type: workflow` and `tags: ["contributing", "onedrive"]`
- **THEN** the page's `tags` array is `["contributing", "onedrive", "workflow"]` after transformation

#### Scenario: Page has no type field

- **WHEN** `addTypeTag` is `true` and the page frontmatter has no `type` field
- **THEN** the `tags` array is not modified

#### Scenario: addTypeTag disabled

- **WHEN** `addTypeTag` is `false` and the page frontmatter has `type: concept`
- **THEN** the `tags` array is not modified

---

### Requirement: repo:// links in the markdown body are rewritten to GitHub URLs

The plugin SHALL rewrite markdown link hrefs that use the `repo://` scheme to fully-qualified GitHub URLs by prepending the configured `repoBase` and URL-decoding the path.

`repo://` links are used by OpenWiki to reference source files in the underlying repository (e.g. Word documents, PDFs, spreadsheets) that the wiki page was generated from.

The rewriting rule is:
```
repo://<url-encoded-path>  →  <repoBase>/<decoded-path>
```

Only links in the markdown body (rendered as `<a>` elements) are rewritten. `repo://` references in frontmatter `sources[]` are not modified.

#### Scenario: Body contains a repo:// link

- **WHEN** the markdown body contains `[AI Maturity Framework](repo://Program%20management/Overview/AI%20Maturity%20Framework.docx)`
- **THEN** the rendered href is `https://github.ibm.com/org/repo/blob/main/Program management/Overview/AI Maturity Framework.docx`

#### Scenario: Body contains multiple repo:// links

- **WHEN** the markdown body contains several `repo://` links
- **THEN** each is independently rewritten using the same `repoBase`

#### Scenario: Body contains no repo:// links

- **WHEN** no `repo://` links are present in the markdown body
- **THEN** all existing links are left unchanged

#### Scenario: repoBase has a trailing slash

- **WHEN** `repoBase` ends with `/`
- **THEN** the resulting URL does not contain a double slash between the base and the path

---

### Requirement: Non-OKF pages are passed through unmodified

The plugin SHALL leave pages that contain no OKF frontmatter fields unchanged — all three transforms are no-ops when their triggering signals are absent.

#### Scenario: Page with no OKF frontmatter

- **WHEN** a page has no `type`, no `repo://` links, and no broken-link comments
- **THEN** the page content and frontmatter are identical before and after transformation
