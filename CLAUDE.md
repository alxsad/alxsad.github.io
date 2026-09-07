# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Hugo static site: a one-page CV for Alex Davidovich, deployed to GitHub Pages at
https://davidovi.ch (custom domain, see `static/CNAME`). There is no application code,
no tests, no linter, and no package manager — the whole site is `data/content.yaml`
rendered by a vendored theme.

## Commands

```sh
git submodule update --init --recursive   # required after clone — theme is a git submodule
make serve                                # hugo server -D  → http://localhost:1313
make build                                # hugo --minify   → ./public (gitignored)
```

Requires **hugo extended** (the theme compiles SCSS); CI pins 0.165.0.

`themes/simple-cv/` is empty until the submodule is initialized. Hugo does **not**
fail in that state — it builds a site with no `index.html` and only warns
("found no layout file for html for kind home"). If a build produces no homepage,
check the submodule first.

## The theme is an abandoned upstream submodule

`themes/simple-cv/` points at `github.com/hootan09/simple-cv`, pinned to commit
`32a2ca6`. It targets a much older Hugo, so parts of it break on current releases.

**Never edit files inside `themes/simple-cv/`** — it is a separate repository and
changes there are not committed with the site. Hugo resolves a project file before the
theme's file of the same path, so override instead:

- `layouts/partials/head.html` — project override that drops the theme's Google
  Analytics block. The theme calls `_internal/google_analytics_async.html` and
  `.Site.GoogleAnalytics`, both removed from Hugo; Go parses templates whole, so that
  block breaks the build even though its `if` is false. Keep this file in sync if the
  submodule is ever updated.

One known deprecation is still unfixed: the theme's `layouts/index.html` and
`assets/scss/main.scss` use `.Site.Data` (deprecated in Hugo 0.156, `hugo.Data` is the
replacement). It only warns today, but will break when Hugo removes it. Fixing it means
overriding both files — i.e. forking most of the theme into the project.

## Architecture

**Content is data, not Markdown.** There is no `content/` directory. The entire page is
built from `data/content.yaml` by `themes/simple-cv/layouts/index.html`, which resolves
`(index .Site.Data .Site.Language.Lang).content` and falls back to `.Site.Data.content`
(so a translation would live in `data/<lang>/content.yaml`). **To change anything the
visitor reads, edit `data/content.yaml` — nothing else.**

Each top-level key maps to one partial in `themes/simple-cv/layouts/partials/`, and every
partial is wrapped in `{{ if .Key }}` — omitting a key simply drops that section:

| Key | Partial | Fields consumed |
|---|---|---|
| `Dir` | (layout + SCSS) | `ltr` / `rtl`; also sets `$direction` in SCSS, which switches the body font |
| `BasicInfo` | `_avatar`, `_contacts` | `FirstName`, `LastName`, `Photo`, `Contacts[].{Icon,Info,Link}` |
| `Profile` | `_profile` | free text (rendered `safeHTML \| emojify`) |
| `Personal` | `_personal` | list of `{Gender, BirthDate, Status?, MaritalStatus?}` |
| `Education` | `_education` | `{Degree, FieldOfStudy, NameofSchool, Location, Date, RelevantCourses?[]}` |
| `Skills` | `_skills` | list of free strings |
| `Experience` | `_experience` | `{Employer, Place, Positions[].{Title, Date, Details[], Badges[]}}` |

Gotchas in that mapping:

- Contacts with `Link: true` are rendered as `href="https://{{ .Info }}"`, so `Info` must
  be host+path **without** a scheme (`linkedin.com/in/alxsad`).
- `Icon` is a raw Font Awesome 5 class string (`fas fa-envelope`), loaded from a CDN.
- Section headings (`Work Experience`, `Skills`, …) come from the theme's
  `i18n/en.toml`, not from the data file.
- `BasicInfo.Photo` is a path relative to the site root; the file lives in
  `static/img/` (`static/img/avatar.jpg` → `img/avatar.jpg`).

**Styling is configured, not hand-written.** `themes/simple-cv/assets/scss/main.scss` is
run through `resources.ExecuteAsTemplate` before SCSS compilation, so its color variables
read from `[params]` in `config.toml`: `colorLight`, `colorDark`, `colorPageBackground`,
`colorPrimary`, `colorIconPrimary`, `colorIconSecondary`, `colorIconBackground`,
`colorLeftColumnBackground`, `colorRightColumnHeadingText`, `colorRightColumnBodyText`,
`pages`. `config.toml` has **no** `[params]` block, so the theme's `| default` values
apply — restyle by adding `[params]` there, never by editing the submodule.
`enableMetaTags = true` additionally turns on the OpenGraph/description tags in `head.html`.

`config.toml` sets `disableKinds` to switch off pages, sections, taxonomies, RSS and
sitemap: it is a single page, and without that Hugo emits unused `tags/`, `categories/`,
`index.xml` and `sitemap.xml` and warns about missing taxonomy layouts.

## Deployment

`.github/workflows/hugo.yml` runs on every push to `master` (and `workflow_dispatch`):
installs hugo_extended `HUGO_VERSION`, checks out with `submodules: recursive`, builds with
`--minify --baseURL "<pages base_url>/"` — **overriding the `baseURL` in `config.toml`** —
uploads `./public`, then deploys via `actions/deploy-pages`.

- Keep `HUGO_VERSION` in the workflow matching the Hugo used locally; the theme is
  version-sensitive (see above), so a silent drift is how this site breaks.
- The download URL uses the modern asset name (`hugo_extended_<v>_linux-amd64.deb`).
  Older Hugo releases used `..._Linux-64bit.deb`; changing the version may require
  changing the filename too.
- `public/` and `resources/` are gitignored build output — CI rebuilds them from source.
- The custom domain is part of the build: `static/CNAME` → `public/CNAME`. Hugo copies
  only `static/` into the output, so a `CNAME` anywhere else never reaches the site.
