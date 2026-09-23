# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Helen Kowalewska's academic site. Hugo static site on a hand-written theme, deployed to GitHub
Pages.

## Commands

Tasks live in `poe_tasks.yaml` and run with [poethepoet](https://poethepoet.natn.io)
(`uv tool install poethepoet`). `poe` on its own lists them.

```sh
poe up        # dev server, drafts included, http://localhost:1313
poe down      # stop it
poe build     # production build into public/
poe clean     # remove public/ and resources/
poe staging   # build for the staging host and rsync it there
```

Every task shells out to the pinned Hugo image in `docker-compose.yml`, so local output matches CI.
The image entrypoint is Hugo itself, so subcommands and flags are passed without a leading `hugo`,
which is why the underlying command reads `docker compose run --rm server --gc --minify`.

The system `hugo` works too and is fine for a quick check, but prefer Docker so local output
matches CI. This repo was pinned to 0.116.1 for years on the belief that the vendored theme broke on
anything newer. It did not; the theme has since been removed entirely.

There are no tests or linters. CI is a single workflow, see Deployment.

## Deployment

Pushing to `master` deploys. `.github/workflows/deploy.yml` builds with the pinned Hugo image and
publishes to GitHub Pages via `actions/deploy-pages`. CI commits nothing, and no build output lives
in the repo.

The Pages source is **GitHub Actions**, not a branch. The custom domain `helenkowalewska.uk` is set
in repo settings and reasserted on every build by `static/CNAME`.

`public/` is gitignored local build output. Delete it freely.

### Staging

`poe staging` builds with `--environment staging` and rsyncs `public/` to the home server, which
serves it at `https://helen-staging.kowfam.uk` for Helen to review before anything goes live.

Two things make a staging build differ from production, and nothing else does. `config/staging/`
overrides `baseurl`, and `layouts/_partials/head.html` adds `noindex, nofollow` for any
non-production environment, which matters because the staging vhost is real HTTPS on a resolvable
name. CSS minification and fingerprinting are gated on `hugo.IsServer`, not `hugo.IsProduction`,
specifically so staging is otherwise byte-identical to what ships.

The rsync target lives in `.env`, which is gitignored because this repo is public. Copy
`.env.example` to start. The server side is one nginx container defined in the `home-infra` repo at
`ansible/roles/compose/files/services/helen-staging/`; its vhost and TLS are derived automatically
from the compose service key and the published port.

The site was previously published by hand into a `public/` submodule pointing at a second repo.
That output history is preserved on the `legacy-output` branch, which doubles as the rollback
target: set Settings -> Pages -> Source back to "Deploy from a branch" and pick `legacy-output`.

## Architecture

**No third-party theme.** The site runs on roughly a dozen hand-written templates in `layouts/`.
The Wowchemy/Academic theme it used to vendor is gone, along with its widget system, client-side
search, isotope filters and Bootstrap. Nothing external is left that can rot.

**Config** is split across `config/_default/`. `config.toml` holds site settings and `ignoreFiles`;
`params.toml` holds everything the templates display, and every key in it is used somewhere;
`menus.toml` and `languages.toml` do what their names suggest. `config/staging/` overrides only
`baseurl`, and is merged over `_default/` when Hugo runs with `--environment staging`.

**Templates** use Hugo's current layout structure: `baseof.html`, `home.html`, `section.html` and
`page.html` at the root of `layouts/`, partials in `layouts/_partials/`, and the Markdown link render
hook in `layouts/_markup/`.

`_partials/entry.html` is the one to understand. It renders a single row of the year-rail ledger and
is used by every listing on the site, working out the rail label and link chips from `.Section`. Add
a content section and it needs a branch there; that is the only place section-specific logic lives.

**Content sections** are `publication`, `media`, `talk`, `project`, `take-part`, `authors` and
`post` (retired, see below). Each page is a page bundle at `content/<section>/<slug>/index.md`, and
each section has an `_index.md` supplying the list page title and intro. `content/teaching/` holds
only PDFs linked from the homepage.

Front matter is deliberately minimal — a publication is about eight lines. Three field names are
chosen to dodge Hugo reserved keys and must not be renamed back: **`link`** (not `url`, which
overrides the page URL), **`pubtype`** (not `type`, which drives layout lookup) and **`medium`** (not
`kind`, removed as a front-matter key in Hugo 0.144).

**Hand-maintained lists live in `data/`**, not in content. `grants.yaml` and `teaching.yaml` are read
by `home.html` and `teaching/section.html`; `statements.yaml` holds the hero questions. Adding a
grant means adding three lines of YAML.

**The hero rotates.** Every question in `data/statements.yaml` renders into the `h1`, stacked in one
grid cell so the hero reserves the height of the tallest and the rotation cannot move the bottom rule
or the anchored portrait. `hero-rotate-js.html` steps through them once, seven seconds apart, and
stops on the last; it moves `is-current` and `aria-hidden` together so the heading always has exactly
one accessible name. It does nothing at all under `prefers-reduced-motion`, which the stylesheet's
global block cannot cover because that kills transitions but not timers. Without JavaScript the first
question simply stays.

**Styling** is one hand-written file, `assets/css/main.css`, run through `resources.ExecuteAsTemplate`
so it can interpolate the self-hosted font URLs from `assets/fonts/`. No Sass, no Tailwind, no build
step beyond Hugo itself. Colours are `oklch` custom properties with light and dark defined on
`:root`, a `prefers-color-scheme` block and an explicit `[data-theme]` block so the toggle wins in
both directions.

Two accents, `--accent-1` (teal) and `--accent-2` (ochre), alternate section by section purely for
rhythm. They carry no meaning. An earlier version tried to make teal mean "scholarly" and ochre mean
"public engagement", but with five homepage sections of which three are neither, it only ever read as
a stray colour.

Components never name an accent. They read `var(--accent)`, which `:root` sets to the first and a
section flips with `.accent-alt`. Adding a section is one class, not a set of override rules.

Minification and fingerprinting are gated on `hugo.IsServer`, not `hugo.IsProduction`, so only the
dev server skips them and a staging build is otherwise byte-identical to production.

**External links open in a new tab.** `_partials/extlink.html` emits the attributes when a URL's host
differs from `site.BaseURL`; it is called from the link render hook, the entry partial, the single
page template and the footer. Its output must be piped through `safeHTMLAttr`, or Go's contextual
autoescaper emits `ZgotmplZ` instead of the attributes.

Helen's profile lives in `content/authors/helen/`, her portrait at `assets/img/portrait.jpg`. The CV
filename carries a date, so replacing it means updating `params.toml` too.

## Conventions

Work is committed **directly to `master`**. This is a single-maintainer site with a linear history
and no PR workflow, so it is a deliberate exception to any general "branch first" rule. Do not
open a branch unless asked.

Site content is UK academic writing, so use British English spelling throughout.

`.editorconfig` sets 2-space indents, LF endings and a final newline. TOML wraps at 100 columns,
Markdown keeps trailing whitespace, and shortcode HTML gets no final newline.
