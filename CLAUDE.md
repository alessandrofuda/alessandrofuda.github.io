# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Local Development

```bash
# Install dependencies
gem install github-pages

# Ensure gem binaries are in PATH (add to ~/.bashrc for persistence)
export PATH="$HOME/gems/bin:$PATH"

# Serve locally at http://127.0.0.1:4000/
jekyll serve
```

Deployment is automatic: every push to `master` triggers a GitHub Pages build.

## Architecture

This is a **Jekyll Now** (v1.2.0) blog — a minimal Jekyll theme without an external theme gem (styles are embedded in the repo).

- `_config.yml` — site identity, social links, Jekyll plugins config
- `_layouts/` — `default.html` is the master wrapper; `post.html` and `page.html` extend it
- `_includes/` — reusable fragments: `meta.html` (SEO), `analytics.html`, `svg-icons.html`, `disqus.html`
- `_sass/` — Sass partials imported by `style.scss`; `_variables.scss` holds colors/fonts/breakpoints
- `_posts/` — blog posts named `YYYY-M-D-Title.md` with front matter (`layout`, `title`)
- Static pages (`about.md`, `404.md`) live at the root alongside `index.html`

## Content

- **New post**: create `_posts/YYYY-M-D-Title.md` with `layout: post` and `title:` in front matter
- **New page**: create `name.md` at root with `layout: page`, `title:`, and `permalink:` in front matter
- **Markdown**: Kramdown with GFM; syntax highlighting via Rouge (Solarized Light theme)
- **Plugins active**: `jekyll-sitemap` (auto-generates `sitemap.xml`), `jekyll-feed` (auto-generates `feed.xml`)

## Social Links & Optional Features

Footer social icons are toggled in `_config.yml` under `footer-links`. Google Analytics and Disqus comments are opt-in via `google_analytics` and `disqus` fields in `_config.yml`.
