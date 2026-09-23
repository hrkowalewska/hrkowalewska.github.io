# helenkowalewska.uk

Helen Kowalewska's academic site. A Hugo static site on a hand-written theme, deployed to GitHub
Pages.

## Running it locally

Tasks are defined in `poe_tasks.yaml` and run with [poethepoet](https://poethepoet.natn.io)
(`uv tool install poethepoet`). Run `poe` on its own to list them.

```sh
poe up        # dev server on http://localhost:1313, drafts included
poe down      # stop it
poe build     # production build into public/
poe staging   # publish to the staging site for review
```

Everything runs through a pinned Hugo container, so you need Docker but not Hugo itself. Local
output matches CI exactly.

## Adding content

Each item is a folder under `content/` containing an `index.md`. Copy the nearest existing one and
edit it; the front matter is short and the fields are named for what they are.

A publication looks like this in full:

```yaml
---
title: "Gendered Employment Patterns: Women's Labour Market Outcomes across 24 Countries"
date: 2023-01-19T00:00:00+01:00
authors: [helen]
pubtype: article
venue: "Journal of European Social Policy"
doi: "10.1177/09589287221148336"
link: "https://journals.sagepub.com/doi/10.1177/09589287221148336"
abstract: |-
  A comparative account of how women's labour market outcomes cluster…

  Leave a blank line to start a new paragraph.
---
```

`pubtype` is one of `article`, `working-paper`, `report`, `thesis`, `chapter`, `book` or
`conference-paper`. It sets the label in the left-hand rail.

A media item is shorter. `medium` is one of `press`, `writing`, `broadcast`, `evidence` or `talk`,
and sets the rail label. Add an optional `headline` when the source's own title is too long or too
tabloid to use as-is; `title` is then what the site shows and `headline` records the original.

```yaml
---
title: "Why money and power affects male self-esteem"
date: 2025-05-20T00:00:00+01:00
medium: press
link: "https://www.bbc.co.uk/future/article/20250519-why-money-and-power-affects-male-self-esteem"
summary: "[bbc.co.uk](https://www.bbc.co.uk/…), May 2025: Interviewed on…"
---
```

Sections sort by `date`, newest first, and group by year. Nothing else needs updating when you add
an item; the listings and the homepage pick it up automatically.

Roles, grants and teaching are not content pages. They live as short YAML lists in `data/`.

## Reviewing changes

`poe staging` builds the site and publishes it to the home server at
`https://helen-staging.kowfam.uk`, so changes can be looked at before they go live. It builds from
your working copy rather than from git, so uncommitted edits show up there too.

The staging build is identical to production apart from the address and a `noindex` tag that keeps
it out of search results.

## Publishing

Pushing to `master` builds and deploys to GitHub Pages automatically. No build output lives in the
repo, and CI commits nothing.

`CLAUDE.md` has the architecture notes, the deployment details and the handful of gotchas worth
knowing before changing templates.
