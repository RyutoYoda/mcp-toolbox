# Link forms, path mapping, and the traps

Background for the `docsite-link-sweep` skill: how a file path in `docs/en` becomes a URL on
`mcp-toolbox.dev`, and the structures that make a naive link checker wrong on this repo. Config
values move, so verify anything load-bearing against `.hugo/hugo.toml` and the workflow files.

## File path to URL

`docs/en` is mounted at the site root:

```toml
# .hugo/hugo.toml
[[module.mounts]]
  source = "../docs/en"
  target = 'content'
```

With `defaultContentLanguage = "en"` and `defaultContentLanguageInSubdir = false`, **no URL has an
`/en/` or `/docs/` segment**. `uglyURLs` is unset, so URLs are pretty.

| File | URL |
| --- | --- |
| `docs/en/documentation/foo.md` | `/documentation/foo/` |
| `docs/en/documentation/_index.md` | `/documentation/` |
| `docs/en/integrations/postgres/source.md` | `/integrations/postgres/source/` |

The `aliases:` frontmatter on the Knowledge Catalog pages confirms this: it is written
site-absolute with no `/en/` prefix.

## The canonical link form

File-relative, with the `.md` extension. `DEVELOPER.md` is authoritative; the short version is that
one string satisfies both checkers, because lychee resolves it as a filesystem path and Hugo
resolves it to the pretty URL.

- Directory-style links (`](../mcp-apps/)`) render correctly in Hugo and fail lychee.
- Site-absolute links (`](/reference/cli/)`) pass lychee and break on versioned deploys.

Counts drift, so run the greps in the skill rather than trusting a number here. Rough orientation
at the time of writing: file-relative `.md` dominates by roughly 4:1 over directory-style,
site-absolute links number in the single digits, `{{< relref >}}` is not used at all, and the only
two `{{< ref >}}` usages are in the Firestore validate-rules page.

## Why site-absolute links leak across versions

Hugo does **not** prefix a hand-written absolute markdown path with `baseURL`, and each deploy sets
a different base:

| Deploy | `HUGO_BASEURL` |
| --- | --- |
| push to `main` | `https://mcp-toolbox.dev/dev/` |
| release | `https://mcp-toolbox.dev/<version>/` and `https://mcp-toolbox.dev/` |
| PR preview | `/` |

So `](/reference/cli/)` on the `/dev/` build resolves to `mcp-toolbox.dev/reference/cli/`, the
*latest-release* docs rather than dev. Every archived version build leaks the same way. Absolute
`https://mcp-toolbox.dev/...` self-links have the mirror-image problem: an archived `/v1.5.0/` page
silently links forward to current docs.

No markdown file hardcodes a versioned path today. The risk is the opposite: unversioned absolute
links that always mean "latest".

## Links that exist only in rendered HTML

These shortcodes build `<a href>` from `.RelPermalink`. The markdown contains no link at all, so a
grep-based checker sees a page with no outbound links, and sees nothing when a target is deleted.
The shortcode just renders one row fewer.

| Shortcode | Roughly how widely used | What it generates |
| --- | --- | --- |
| `{{< compatible-sources >}}` | ~300 pages | source pages compatible with a tool |
| `{{< list-tools >}}` | ~50 pages | the tool pages under a source |
| `{{< samples-gallery >}}` | 1 | every page with `is_sample: true` |
| `{{< list-prebuilt-configs >}}`, `{{< list-db >}}` | 1 each | directory listings |
| `{{< include >}}`, `{{< regionInclude >}}` | ~9 | another file's body, links and all |

Also rendered rather than written:

- **`shared_tools` frontmatter** injects a managed database's inherited tool links, via
  `.hugo/layouts/partials/hooks/body-end.html`.
- **`.hugo/layouts/docs/redirect.html`** renders a meta-refresh page from `external_url`.
- **`llms.txt` and `llms-full.txt`** emit absolute `Permalink`s for every page, baking in the
  deploy-time baseURL. `CLAUDE.md` requires hand-updating the Diátaxis section in both layouts when
  a top-level docs section is added.

Building the site and crawling `public/` is the only way to check any of this.

## Traps

- **`ignoreFiles`.** `.hugo/hugo.toml` lists several `quickstart/` paths. They exist on disk, so
  lychee resolves links to them happily, but Hugo never builds them into pages. The link 404s on
  the live site and passes CI forever.
- **Missing `type: docs`.** A section `_index.md` without it gets the Docsy default layout instead
  of `.hugo/layouts/docs/section.html`, which is what lists child pages. The page renders with
  chrome and zero child links: a dead end containing no broken link. This hit 47 integration index
  pages (issue #3752, fixed in `59fb2c42180`).
- **Leaf-to-section promotion.** When a page becomes a directory, `foo.md` turns into
  `foo/_index.md` and every link naming the old leaf breaks. See `c63efb0568c`, which rewrote
  `../configure.md` to `../configuration/_index.md`.
- **Two `getting-started` directories.** `docs/en/getting-started/` and
  `docs/en/documentation/getting-started/` both exist, so `../getting-started/` resolves
  differently depending on the linking file's depth. Always resolve against the real file.
- **Aliases get forgotten.** The Dataplex to Knowledge Catalog rename added `aliases:` across ~23
  files, but later moves (the Groups docs relocation, the SDK page redirects) added none. Treat a
  missing alias on a rename as an oversight to raise, not a decision already made.

## Historical failure modes worth grepping for

```bash
git log --oneline -i --grep='broken link' --grep='dead link' --grep='fix.*link' --grep='docsite link' -- docs/
```

Recurring shapes: docs reorgs that move directories, tool and source page renames, absolute GitHub
URLs replaced by relative paths (`adc67c12a94`), and site-absolute paths converted to relative
(`3f400efdaa6`).
