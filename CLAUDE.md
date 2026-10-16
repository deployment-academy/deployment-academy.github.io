# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is a Hugo static blog (deployment.properties). There are no tests or linters; a successful Hugo build is the check.

## Commands

Hugo is not installed locally. Run everything through Docker using `docker-compose.yml`, whose `hugo` service uses `hugomods/hugo:git-0.118.2` (the same Hugo version CI pins, 0.118.2, non-extended).

- First clone: `git submodule update --init --force` (the theme is a submodule).
- Dev server with live reload at http://localhost:1313, includes drafts and future-dated posts: `docker compose up`
- One-off production-style build (excludes drafts, like CI), written outside the repo because `public/` is not gitignored: `docker run --rm -v "$PWD":/src -w /src hugomods/hugo:git-0.118.2 sh -c 'hugo -d /tmp/out && grep ... /tmp/out/index.html'`. The output lives only inside the container, so inspect it in the same `sh -c`.
- New post: `docker compose run --rm hugo hugo new posts/<category>/<slug>.md`. The README shows `posts/name.md`, but posts actually live in category subdirectories.

## Architecture

- Theme: Mainroad, a git submodule at `themes/mainroad` (https://github.com/vimux/mainroad). Never edit the submodule. Override or add templates under the site's `layouts/` (Hugo prefers site files over theme files).
- Site overrides that exist:
  - `layouts/shortcodes/notice.html`: the `notice` shortcode, params `type` (warning|info|note|tip), `id`, `title`. Usage: `{{< notice type="tip" id="x" title="T" >}}...{{< /notice >}}`
  - `layouts/partials/widgets/highlights.html`: custom sidebar widget.
- Sidebar: the theme's `partials/sidebar.html` renders each widget name as `partials/widgets/<name>.html`, so a new widget is just a new partial there.
- `config.toml` `[Params.sidebar]`: home and list pages show a right sidebar; single pages (`single = false`) have none. The default widget list for home and list pages is `["search", "highlights", "social"]`. `search` is the theme's Google site search; `ddg-search` and `recent` are available but unused there.
- A page can override widgets with `sidebar`/`widgets` front matter. Only `content/about/_index.md` does (search, highlights, recent, social). Posts must not set `sidebar` or `widgets`.
- Highlights widget: lists regular pages in `mainSections` (`posts`) with front matter `highlight: true`, newest first, up to `highlights_num` (config, 5). It renders nothing if none are flagged. Override the title via `Params.widgets.highlights_title`.
- `config.toml` wires empty `static/css/custom.css` and `static/js/custom.js` via `customCSS`/`customJS`. `static/CNAME` sets the custom domain.
- `enableGitInfo = true`, so Hugo takes `.Lastmod` from git history by default. That is why CI checks out with `fetch-depth: 0`.

## Content

- Posts: `content/posts/<category>/<slug>.md`. Category directories are `eng`, `networking`, `devsecops`, `k8s-ops`, `hugo`.
- About page: `content/about/_index.md`.
- URL form: `https://deployment.properties/posts/<category>/<slug>/`.
- Archetype `archetypes/default.md` front matter: title, description ("changeme" placeholder to replace), date, lastmod, `draft: true`, tags.
- Optional fields in use:
  - `toc: true` for long posts.
  - `categories: ["Key Concepts"]` only for that named series. Categories are a separate taxonomy from tags; don't invent new ones.
  - `highlight: true` to feature the post in the widget.
- `<!--more-->` marks the teaser cutoff.
- Publish or unpublish by flipping `draft` and bumping `lastmod`, nothing else.

## Deployment and git workflow

- `.github/workflows/gh-pages.yml` runs on push to `main` and on pull requests. It builds with `hugo --minify` and deploys `./public` to GitHub Pages (peaceiris/actions-gh-pages) only on `main`. A PR's CI run is therefore the build check.
- `main` is branch-protected. All changes go through a PR from a branch.
- PRs are squash-merged, producing titles like "Add highlights sidebar widget (#72)".
- Before committing, confirm you are on a feature branch and that its PR is still open; after a merge the working copy is often back on `main`.

## Writing style guide

The style guide for posts (voice, tone, punctuation, structure) previously lived in this file. Recover it with `git show 92c9991:CLAUDE.md`.
