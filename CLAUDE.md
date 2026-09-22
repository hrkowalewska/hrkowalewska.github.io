# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Helen Kowalewska's academic site. Hugo static site, Wowchemy/Academic theme, deployed to GitHub
Pages.

## Commands

Run the dev server through Docker, which pins the only Hugo version this site builds on:

```sh
docker compose up   # hugomods/hugo:0.116.1, drafts included, http://localhost:1313
```

**Do not run the system `hugo`.** It is far newer (0.162.x) and the Academic 4.3.1 theme breaks on
it. `bin/hugo` (a stale 0.55.6 x86_64 binary), `view.sh`, `update_academic.sh` and
`scripts/init_kickstart.sh` are legacy leftovers from the theme's own tooling. Ignore them.

Production build, same pinned image:

```sh
docker compose run --rm server hugo --gc --minify
```

There are no tests or linters. CI is a single workflow, see Deployment.

## Deployment

Pushing to `master` deploys. `.github/workflows/deploy.yml` builds with the pinned Hugo image and
publishes to GitHub Pages via `actions/deploy-pages`. CI commits nothing, and no build output lives
in the repo.

The Pages source is **GitHub Actions**, not a branch. The custom domain `helenkowalewska.uk` is set
in repo settings and reasserted on every build by `static/CNAME`.

`public/` is gitignored local build output. Delete it freely.

The site was previously published by hand into a `public/` submodule pointing at a second repo.
That output history is preserved on the `legacy-output` branch, which doubles as the rollback
target: set Settings -> Pages -> Source back to "Deploy from a branch" and pick `legacy-output`.

`netlify.toml` is vestigial, pins an ancient Hugo version, and is not the deploy path.

## Architecture

The theme is **Wowchemy/Academic v4.3.1**, vendored as ordinary tracked files at `themes/academic`.
Never edit it. This repo has no git submodules. `themes/helen-2024` is empty and unused.

**Config** is split across `config/_default/`. `config.toml` holds site settings, taxonomies and
`ignoreFiles`; `params.toml` holds theme options, contact details and `plugins_css`; `menus.toml`
and `languages.toml` do what their names suggest.

**The homepage is widget-driven.** Every file in `content/home/` is a headless widget page.
`widget = "..."` picks which theme widget renders, `weight` orders the section, and `active`
toggles it on or off. Nav entries in `menus.toml` are anchors named after the widget file, so
`#featured` targets `content/home/featured.md`.

**Content sections** are `publication`, `media`, `project`, `talk`, `take-part`, `authors`, and
`post` (retired, see below). Each page is a page bundle at `content/<section>/<slug>/index.md`
with `featured.png` beside it for card thumbnails and social share images, plus `cite.bib` for
publications. Section-level front matter lives in `_index.md` and sets `view:` (1 list, 2 compact,
3 card, 4 citation). `content/teaching/` is the exception. It has no `_index.md` and just holds
PDFs that the Teaching widget links to directly.

**`media` is a custom section the theme does not ship.** It renders only because of three
hand-written overrides. `layouts/section/media.html` is the list page, `layouts/media/single.html`
is the single page, and `layouts/partials/li_compact.html` carries a `media` branch alongside the
theme's built-in types. Adding another custom section needs the same three pieces.

Everything in `layouts/` shadows the matching path under `themes/academic/layouts/`. To customise,
copy the theme file to the same path here and edit the copy. `i18n/en.yaml` overrides theme
strings the same way, and `assets/css/custom.css` is loaded via `plugins_css = ["custom"]` in
`params.toml`.

Helen's profile and CV live in `content/authors/helen/`. The CV filename carries a date, so
replacing it means updating the reference in `_index.md` too.

## The retired News section

The `post` section is switched off in three places at once. Re-enabling it means reversing all of:

1. the `content/post/` entry in `ignoreFiles` in `config/_default/config.toml`,
2. `active = false` in `content/home/posts.md`,
3. the commented-out News block in `config/_default/menus.toml`.

Each of the three carries a comment pointing at the others. Changing only one leaves the widget
rendering empty.

## Conventions

Work is committed **directly to `master`**. This is a single-maintainer site with a linear history
and no PR workflow, so it is a deliberate exception to any general "branch first" rule. Do not
open a branch unless asked.

Site content is UK academic writing, so use British English spelling throughout.

`.editorconfig` sets 2-space indents, LF endings and a final newline. TOML wraps at 100 columns,
Markdown keeps trailing whitespace, and shortcode HTML gets no final newline.
