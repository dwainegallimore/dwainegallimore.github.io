# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is Dwaine Gallimore's personal blog, built with **Jekyll** on top of the **Thinkspace** theme (a minimalist Jekyll theme, vendored directly into this repo rather than pulled in as a remote gem — see `thinkspace.gemspec`). It's deployed via GitHub Pages with a custom domain (`CNAME` → `www.gallimore.dev`).

## Commands

Ruby/Bundler and Jekyll are required (see `Gemfile` / `thinkspace.gemspec`, which pins `jekyll ~> 4.0.0`).

```bash
bundle install                 # install gems (first time / after Gemfile changes)
bundle exec jekyll serve       # run local dev server with live rebuild (default: http://localhost:4000)
bundle exec jekyll build       # build the static site into _site/
```

CI build/verification (mirrors `.travis.yml` / `script/cibuild.sh`):

```bash
bundle exec jekyll build
bundle exec htmlproofer _site --disable-external --allow-hash-href
```

There is no test suite beyond `html-proofer` link/markup checking of the built `_site/` output — there are no unit tests to target individually.

## Architecture

Standard Jekyll layout/include/data composition, with no build tooling (no npm/webpack/gulp) — Sass is compiled directly by Jekyll via the `sass` config in `_config.yml`.

- **`_config.yml`** — site-wide settings (title, description, permalink pattern `/:year/:title/`, navbar entries, Sass source dir, plugin list). Changes here often require a full `jekyll build`/`serve` restart (Jekyll doesn't hot-reload config).
- **`_posts/`** — blog posts, filename convention `YYYY-MM-DD-title.md`. Each post's front matter uses `layout: post`, `title`, `description`, `comments`, `keywords`. Comments are globally disabled (`comments_enabled: false` in `_config.yml`), so per-post `comments:` front matter is currently inert.
- **`_layouts/`** — page templates:
  - `default.html` is the outer shell (wraps everything in `compress` layout, includes `head`/`header`/`footer`/`foot`).
  - `compress.html` is a third-party whitespace-compression layout (vendored, not meant to be edited).
  - `page.html` / `post.html` extend `default` for static pages vs. blog posts respectively.
- **`_includes/`** — partials pulled into layouts (`head.html`, `header.html`, `nav.html`, `intro.html`, `footer.html`, `foot.html`, `pagination.html`, `post-share.html`, `comments.html`).
- **`index.html`** — the home page; iterates `site.posts` and groups them into `<h2>` year headers by comparing each post's year to `post.next`'s year (Liquid loop, oldest-to-newest `.next` semantics — easy to get backwards when editing).
- **`about.md`** — a top-level page (`layout: page`) rendered at `/about/`.
- **`assets/scss/`** — Sass source, compiled by Jekyll (`sass.sass_dir: assets/scss`, `style: compressed`). `main.scss` is the entry point that imports `bourbon/`, `base/*`, `_variables.scss`, `_layout.scss`, `_syntax-highlighting.scss`. The `bourbon/` directory is the vendored Bourbon Sass mixin library — treat it as third-party, not app code.
- **`assets/css/`** — pre-built/vendored CSS (Font Awesome) served as-is, not compiled from Sass.
- Adding a navbar entry requires both a page in the repo root (with matching `permalink`) and an entry under `navbar:` in `_config.yml`.

## Conventions

- New blog posts go in `_posts/` following the `YYYY-MM-DD-title.md` naming convention Jekyll requires for date parsing/permalinks.
- Keep front matter fields consistent with existing posts (`layout`, `title`, `description`, `comments`, `keywords`).
- `_site/`, `.sass-cache/`, `.bundle/`, and `Gemfile.lock` are gitignored build artifacts — don't commit them.
