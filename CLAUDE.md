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

`_partials/stack.html` emits the same markup from a `data/` map rather than a Page, so every dated
list on the site shares one set of CSS rules. The teaching, awards and experience lists had a
quieter treatment of their own for a while and it drifted. The fix was to delete it rather than
keep the two in step.

**A row's title only links when there is something to see.** Every publication, media and talk page
has an empty body, so the single page shows less than the row you clicked from. Those three sections
are flat unless an item sets `show_page: true`, which `_partials/showpage.html` decides. Every other
section links as before, and that is a section rule rather than "does the page have a body" because
two project pages are empty too and their chip is the only route to the project site.

A data row has no page of its own, so it names one instead. An optional `page` in `teaching.yaml`,
`grants.yaml` or `experience.yaml` is where the title goes, which is nowhere unless it is set. Do
not confuse it with `link`, which is a chip out to somewhere else.

`showpage.html` also drives `noindex` in `head.html` and the filter in `layouts/sitemap.xml`, so an
unlinked page is not advertised to search engines either. Setting `show_page` puts an item back in
the listing and the sitemap together. That is the whole reason the rule lives in a partial. The
sitemap template is Hugo's default plus that one condition.

**Content sections** are `publication`, `media`, `talk`, `project`, `take-part`, `authors` and
`post` (retired, see below). Each page is a page bundle at `content/<section>/<slug>/index.md`, and
each section has an `_index.md` supplying the list page title and intro. `content/teaching/` holds
only PDFs linked from the homepage.

Front matter is deliberately minimal — a publication is about eight lines. Three field names are
chosen to dodge Hugo reserved keys and must not be renamed back: **`link`** (not `url`, which
overrides the page URL), **`pubtype`** (not `type`, which drives layout lookup) and **`medium`** (not
`kind`, removed as a front-matter key in Hugo 0.144).

Three more are optional. **`show_page`** opts a thin publication, media or talk item back into
being linked and indexed, **`abstract`** expands in place in the listing, and **`pdf`** names a file
in the page bundle and renders it as a chip, the way a talk carries its `slides`. Carrying the file
beats linking someone else's copy, which is how the 2018 report ended up pointing at a 404.

`abstract` is not publication-only. A seminar carries one and a podcast carries its episode notes,
so `entry.html` works out what the disclosure calls itself alongside the rail label, in the same
section branch. A publication or a talk says "Abstract", anything else says "Description". It could
not be named `summary`, which is the one-line description every row already has.

Those reserved-key rules are front matter only. Files under `data/` are plain maps Hugo never
interprets, so `teaching.yaml` and `grants.yaml` can use `kind` for the rail label without trouble.

**Hand-maintained lists live in `data/`**, not in content. `teaching.yaml` and `grants.yaml` each
feed their own homepage section and their own page, `/teaching/` and `/awards/`; `experience.yaml`
renders as Experience on the About page; `statements.yaml` holds the hero questions. Adding a grant
means adding four lines of YAML.

Experience is gated on `show_experience` in Helen's front matter, so a co-author's page does not
list her posts under their name. The portrait is page-specific for the same reason.

`grants.yaml` mixes funding with honours, which is why its heading is "Grants and awards" rather than
naming one or the other, and why each entry carries a `kind` naming which it is. Teaching uses the
same field for her role, so the titles are bare unit names. The caller slices the data and
`stack.html` does the markup, so callers pass nothing but a list.

An entry may carry `end` to show a span. Still running, it is a trailing dash on the year, "2025–".
Ended, it is a "to 2022" label under the start year, because a closed span will not fit on one line
at the size the rail sets a year in. That difference matters, because the label slot is also where
`kind` goes: an open span can carry a role and a closed one cannot, and `kind` wins if both are
set.

**The hero rotates.** Every question in `data/statements.yaml` renders into the `h1`, stacked in one
grid cell so the hero reserves the height of the tallest and the rotation cannot move the bottom rule
or the anchored portrait. `hero-rotate-js.html` steps through them once, seven seconds apart, and
stops on the last; it moves `is-current` and `aria-hidden` together so the heading always has exactly
one accessible name. It does nothing at all under `prefers-reduced-motion`, which the stylesheet's
global block cannot cover because that kills transitions but not timers. Without JavaScript the first
question simply stays.

**The hero regroups below 52rem.** Stacked in markup order it read badly: the eyebrow landed under
the portrait and captioned it, and her name sat a screen below past the whole question. So
`.hero-intro` becomes a grid there and `order` puts the portrait, her name and her title together as
one centred block, with the question following and left alone. Above the breakpoint every `order` is
inert and the two-column hero is untouched.

The eyebrow is hidden there rather than moved. It is a wide-screen device, it will not fit on one
line stacked, and wherever it went it read as something's caption. Its one load-bearing half was the
university, and her title names that at every width, written the way the About page writes the same
line. On a wide screen the eyebrow then says it again 282px above. That repetition is deliberate,
chosen over moving it, so do not tidy it away.

**Styling** is one hand-written file, `assets/css/main.css`, run through `resources.ExecuteAsTemplate`
so it can interpolate the self-hosted font URLs from `assets/fonts/`. No Sass, no Tailwind, no build
step beyond Hugo itself. Colours are `oklch` custom properties with light and dark defined on
`:root`, a `prefers-color-scheme` block and an explicit `[data-theme]` block so the toggle wins in
both directions.

Two accents, `--accent-1` (teal) and `--accent-2` (ochre), carry no meaning. An earlier version
tried to make teal mean "scholarly" and ochre mean "public engagement", but with five homepage
sections of which three are neither, it only ever read as a stray colour.

A section keeps one accent everywhere it appears. `_partials/accent.html` holds the list,
`home.html` reads it for each block and `baseof.html` puts the class on `<main>`, so a whole page
inherits its section's accent and single pages come along for free. The homepage still alternates,
because its running order and that list are arranged to. They used to disagree, which is why
clicking an ochre block landed on a teal page. Reorder the homepage and the list has to follow.

Components never name an accent. They read `var(--accent)`, which `:root` sets to the first and
`.accent-alt` flips. Adding a section is one entry in that list, not a set of override rules.

Minification and fingerprinting are gated on `hugo.IsServer`, not `hugo.IsProduction`, so only the
dev server skips them and a staging build is otherwise byte-identical to production.

**The nav is a drawer below 57rem.** Seven links wrapped onto two rows under the name and left a
sticky 186px masthead on every phone, 28% of an iPhone SE, held all the way down an 18-screen
archive. It is 61px at every width now, and that is the invariant to protect: anything added to the
bar has to keep it on one row. The breakpoint was 52rem until the search trigger arrived, because
904px is the narrowest the bar fits once it carries one. The hero's own 52rem is a separate
question and did not move.

There is one `<nav>` in the markup, not two. `nav-drawer-js.html` adds `.has-drawer` at load and
only then does the CSS switch to drawer mode and reveal the button. With JavaScript off the button
never appears and the nav keeps its wrapping row, so the fallback is what the site did before rather
than a site with no navigation.

Focus is contained by marking `<main>` and the footer `inert`, not by a hand-written trap. The
closed panel is marked `inert` too, and that, rather than `visibility: hidden`, is what keeps it out
of the tab order while it waits off-screen. The CSS version looks tidier and does not work: a
transitioned `visibility` holds its computed value until the transition ends and nothing hidden can
take focus, so the script cannot move focus into a panel already visible on screen. Since that
`inert` must not apply to the ordinary nav row, it is scoped to the breakpoint in the script.

One other trap. `backdrop-filter` is dropped below 52rem, because it makes an element the containing
block for fixed-position descendants, which would strand the panel inside the bar.

**Touch targets live in one `@media (pointer: coarse)` block**, so the wide-screen density is
untouched. Every control was under 44px and two were under the 24px WCAG 2.5.8 floor. Hit area comes
from padding, so nothing changes size to the eye. Two deliberate exceptions. Chips stay at 46x27,
already over the floor, because 44px chips would be the heaviest thing on a page of hairlines. And a
one-line entry title gets a positioned pseudo-element rather than padding, reaching about 28px,
because an inline element's target is its font box and padding it moved every row down.

**Search is a dialog, and a filter.** `layouts/home.json` emits an index of all 86 items, content
pages and the three `data/` files alike, and `search-js.html` fetches it the first time search is
opened and never otherwise. Matching is plain: every token must appear, a title match sorts first.
At this size that beats stemming for both weight and surprise.

Where a result goes is decided in Hugo, not in the script. An item whose own page is worth visiting
links to it, by the same `showpage.html` the listings read; everything else links to its listing
carrying `?q=`, and the filter picks that up on arrival. Writing it once means the rule cannot drift
away from the listings.

That is also why Pagefind is the wrong tool here despite being the obvious one and genuinely
Node-free. It indexes built HTML, so it would find the 53 thin pages that are deliberately unlinked
and return everything twice, and excluding them leaves a result pointing at a whole listing with no
way to reach the row.

A result's rail names the section it will send you to, taken from the nav rather than the section
page's title, because two of them differ and the long forms wrap in a column sized for a year.
Experience is the one exception to `?q=`: it lives on the About page, which is a profile with no
filter, so those link to `#experience` instead.

The same matcher filters a listing in place, which is what `?q=` lands on. It has to hide a year
heading left with nothing under it, of which `/media/` has seven. The box sits with the table rather
than in the page header, ranged right, and its Clear button also strips `?q=` so a reload does not
silently refilter.

**The theme control has three states**, auto, light and dark, cycled in that order. Auto is the
absence of a choice rather than a value: it removes `data-theme` and clears the stored key, so the
`prefers-color-scheme` block governs and the page tracks the system live without a listener. The
label names the current setting and `title`/`aria-label` spell it out; there is no `aria-pressed`,
which is binary. The pre-paint guard in `head.html` only honours a stored `light` or `dark`, so a
stale or junk value falls back to auto rather than stamping an attribute that matches no rule.

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
