---
name: docsite-link-sweep
description: >-
  Sweep the googleapis/mcp-toolbox docs for broken and non-canonical links, report
  every finding with the reason it breaks, and land the fixes for the safe class of
  internal docsite links. Use whenever a maintainer asks for a link sweep, a docs
  health check, or triage of the weekly "Link Checker Report" issue, e.g. "check the
  docs for broken links", "link sweep", "the link checker is red on #3711", "fix the
  dead links in docs/", or after a docs reorg, page rename, or directory move.
  Edits the working tree and leaves a ready commit; never pushes, never opens a PR,
  and never rewrites external links or ambiguous targets on its own.
---

# Docsite Link Sweep (mcp-toolbox)

Two checkers guard these docs and neither is sufficient alone.

- **lychee** is what CI runs. It resolves links as filesystem paths and knows nothing about Hugo.
- **Hugo** is what builds the site. It resolves `.md` links to pretty URLs and generates whole
  classes of links from shortcodes, but never checks an external URL.

A link can pass one and break the other, which is why `.github/workflows/link_checker.yaml` warns
contributors that a fix must satisfy both. So the value of a sweep is not the lychee output, which
anyone can get by re-running the job. It is deciding which failures are real breakage, which are
the two checkers disagreeing, and which single rewrite satisfies both.

## Goal

A report the maintainer can act on in one sitting: every finding sorted into fixed,
needs-a-decision, external, or ignore-worthy, each with its `file:line` and the reason it breaks.
The safe class is already applied and verified, on a branch with a commit message, ready to push.

## Prerequisites

- **Hugo Extended v0.146.0+.** Without a build you cannot see shortcode-generated links.
- **lychee**, if available (`brew install lychee`, or `docker run --rm -v "$PWD:/input"
  lycheeverse/lychee`). Without it, run the grep and build passes only and say so in the report.
- **A clean tree.** Run `git status` first and stop if docs have uncommitted changes. You cannot
  hand back a reviewable commit sitting on top of someone's work in progress.
- **A scope.** Default to `README.md` plus `docs/`, matching the weekly job.

## Workflow

### Step 1: Read the source of truth

Read these live. They move, so this file deliberately does not restate them.

- [`references/DEVELOPER.md`](references/DEVELOPER.md), section "Link Checking and Fixing with
  Lychee": authoritative for the canonical link form, and for when an ignore entry is legitimate.
  Cite it as `DEVELOPER.md` in your output; the symlink path means nothing to a reader.
- [`.lycheeignore`](https://github.com/googleapis/mcp-toolbox/blob/main/.lycheeignore): what is
  already excluded, and why. Every entry carries a comment, and yours must too.
- [`.github/workflows/link_checker.yaml`](https://github.com/googleapis/mcp-toolbox/blob/main/.github/workflows/link_checker.yaml)
  (per PR, changed files only) and
  [`link_checker_report.yaml`](https://github.com/googleapis/mcp-toolbox/blob/main/.github/workflows/link_checker_report.yaml)
  (weekly cron over `README.md` and `docs/`, files an issue titled "Link Checker Report"). Take the
  lychee args from the file rather than from this skill.
- [`references/link-forms.md`](references/link-forms.md): the file-path-to-URL mapping, the
  shortcodes that generate links, and the traps that make a naive checker wrong on this repo.

### Step 2: Reproduce what CI sees

```bash
lychee --quiet --no-progress --exclude '^neo4j\+.*' --exclude '^bolt://.*' README.md docs/
```

For a PR-scoped sweep, restrict to changed files the way the PR job does:

```bash
git diff --name-only --diff-filter=ACMRT origin/main...HEAD -- '*.md'
```

Reproduce before fixing anything. A finding that will not reproduce locally is usually an external
flake or a cache artifact, so it belongs in the external bucket rather than the fix list.

### Step 3: Find what lychee structurally cannot

lychee greps markdown, so these break the site while passing CI:

```bash
# Directory-style relative links: Hugo resolves them, lychee cannot.
grep -rnE "\]\(\.\.?/[^)]*\)" docs/en --include=*.md | grep -vE "\.md(#[^)]*)?\)"

# Site-absolute links: leak out of /dev/ and /vX.Y.Z/ builds to the latest-release docs.
grep -rnE "\]\(/[^)]*\)" docs/en --include=*.md

# Absolute self-links: the same version leak, and they 404 before a page ships to root.
grep -rn "https://mcp-toolbox.dev/" docs/en --include=*.md

# Section indexes missing `type: docs` can render with no child links at all: a dead
# end that contains no broken link. This was issue #3752.
find docs/en -name _index.md -exec grep -L "^type: docs" {} +
```

That last one lists candidates, not defects, and currently matches dozens of files. A page that
sets `no_list: true` or renders its own listing shortcode (`{{< samples-gallery >}}`,
`{{< list-tools >}}`) is deliberate. Open each hit and confirm the built page really has no way
down to its children before reporting it.

Then build and crawl, which is the only way to see shortcode-generated links:

```bash
cd .hugo && hugo --minify --config hugo.cloudflare.toml
lychee --offline --base-url public public   # run `lychee --help`; this flag has been renamed across versions
```

A page whose tool list comes from `{{< list-tools >}}` or `{{< compatible-sources >}}` has no link
in its markdown at all. Delete a target page and the shortcode silently renders one row fewer: no
error, no broken link, just missing content. Only the built HTML shows it.

### Step 4: Classify before you touch anything

Put every finding in exactly one bucket. Only the first is yours to fix.

**Safe to fix.** Mechanical, one correct answer, verifiable both ways:

1. Directory-style relative link, rewritten to the same target in `.md` form.
2. Site-absolute `](/some/path/)`, rewritten to a file-relative `.md` path.
3. Absolute `https://mcp-toolbox.dev/...` self-link, rewritten to a file-relative `.md` path.
4. A moved target where `git log --diff-filter=D --name-only` or `git log --follow` names exactly
   one successor.
5. Anchor drift where the heading was renamed in this repo and the new heading is unambiguous.

**Needs a decision.** Report with a recommendation, but do not apply:

- A target that exists nowhere. Writing the missing page and deleting the link are both defensible,
  so it is the maintainer's call.
- A move with more than one plausible successor, such as a page split in two.
- A link into a path listed in `ignoreFiles` in
  [`.hugo/hugo.toml`](https://github.com/googleapis/mcp-toolbox/blob/main/.hugo/hugo.toml). Those files exist on disk, so
  lychee is happy, but Hugo never builds them into pages and the link 404s on the site.
- A section confirmed to be a dead end for want of `type: docs`. Give the one-line frontmatter
  patch and let the maintainer apply it, since it changes how the whole page renders rather than
  just a link.

**External.** Report the status code and `file:line`, and propose only a fix direction. Guessing a
replacement URL for a dead third-party link swaps a visibly broken link for a plausible-looking
wrong one.

**Ignore-worthy.** Rate-limited, auth-walled, or local-only URLs. Propose a `.lycheeignore` entry
with its explanatory comment. Treat this as a last resort: an ignore entry hides the link from
every future sweep, so anything you ignore is something nobody checks again.

### Step 5: Rewrite to the one form that satisfies both checkers

The canonical form is **file-relative, with the `.md` extension**. lychee finds the physical file
and Hugo resolves it to the pretty URL, from the same string.

Compute the path from the *linking file's* directory, remembering that `docs/en` is mounted at the
site root with no `/en/` and no `/docs/` segment. Two traps produce a wrong-but-plausible path:

- **`_index.md` is the section itself**, not a sibling. A link to a section points at
  `../configuration/_index.md`, so a page promoted from a leaf file to a directory breaks every
  link still naming the old leaf.
- **`getting-started` is ambiguous.** Both `docs/en/getting-started/` and
  `docs/en/documentation/getting-started/` exist, so `../getting-started/` means different things
  at different depths. Resolve it against the actual file, never by matching the name.

### Step 6: Verify every fix both ways

```bash
lychee --quiet --no-progress --offline <the files you changed>   # lychee finds the file
cd .hugo && hugo --environment development                        # Hugo builds with no ref errors
```

Then confirm the rendered `href` in `public/` points where you intended. A relative path can be
wrong by one directory level and still resolve to a real file.

If a fix touches a renamed or moved page, check whether `aliases:` frontmatter belongs at the new
location so the old URL keeps working. The repo has the pattern (see the Knowledge Catalog pages)
but most moves forget it, so raise it explicitly rather than assuming it was decided against.

### Step 7: Land the change, and stop

- Branch `docs/fix-docsite-links`, scoped further if the sweep was.
- One commit: `docs: fix broken docsite links`.
- Safe-class fixes only. No drive-by wording edits, no reformatting, nothing from the decision
  bucket.
- **Stop before pushing.** Report the branch name and the command to run. Never `git push` or
  `gh pr create`.

## Rules

- **Verify both ways or do not ship it.** A change not confirmed against lychee *and* a Hugo build
  is a finding, not a fix.
- **Never rewrite an external link**, and never invent an internal target. A destination that does
  not exist goes in the decision bucket even when the intended page seems obvious.
- **One reason per finding**, with `file:line`. "Broken" is not a reason; "directory-style link,
  lychee cannot resolve it" is.
- **Never ignore an internal link.** Ignore entries are for external URLs only, always with a
  comment.
- **Disclose the edges of the sweep**: the scope, whether you built the site or only grepped, and
  anything you skipped. Silence about coverage reads as "I checked everything."
- **Mark anything unverified** `[UNVERIFIED]` rather than asserting it.

## Output format

```text
## Docsite link sweep: <scope>, <X> findings
Checked: lychee over <scope> | built site: <yes/no> | <N> files changed

**Fixed and verified** (<n>)
| file:line | was | now | why it broke |

**Needs your decision** (<n>)
| file:line | target | the problem | recommendation |

**External, report only** (<n>)
| file:line | url | status |

**Proposed .lycheeignore entries** (<n>)
| pattern | why |

**Structural** (<n>)
- <file>: <e.g. missing `type: docs`, section renders no child links>

**Apply:**
git push -u origin docs/fix-docsite-links
gh pr create --title "docs: fix broken docsite links"
```

- **Empty buckets:** omit them, except **Needs your decision**, which is always stated even when
  empty. "Nothing here needs a judgment call" tells the maintainer the branch is safe to skim
  rather than audit.
- **Large sweeps:** lead with the per-bucket counts and fix in batches, so the maintainer chooses
  how deep to go before reading a hundred rows.
