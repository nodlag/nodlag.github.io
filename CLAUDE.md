# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-page static portfolio site for game developer Javier Garrido (`@nodlag`), hosted on GitHub Pages at https://nodlag.github.io. The site showcases shipped games and skills.

## Stack & build

- **No package.json, no bundler, no tests.** Files are served as-is by GitHub Pages.
- **Tailwind CSS is precompiled to a static file**: `css/tailwind.css` is built from `tailwind.input.css` + `tailwind.config.js` (which scans `index.html` for classes). That file is what `<head>` references. If you add or remove Tailwind classes in `index.html`, regenerate it and commit the result:

  ```sh
  npx -y tailwindcss@3 -c tailwind.config.js -i tailwind.input.css -o css/tailwind.css --minify
  ```

  Do not reintroduce the CDN runtime (`cdn.tailwindcss.com`) — it was removed for render-blocking / LCP reasons.
- **Custom CSS** lives in `css/styles.css` (CSS variables for light/dark theme, project card styles, modal styles, vibrate animation).
- **Custom fonts** are in `fonts/` (`m5x7.ttf` pixel font, `BebasNeue-Regular.ttf`); `@font-face` declarations are at the top of `css/styles.css`.

## Deploy

Pushing to `main` deploys to GitHub Pages — there is no staging environment. Treat every commit to `main` as production.

## Local preview

Open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`) from the repo root. Use a local server rather than `file://` when testing anything that may be affected by CORS (fonts, iframes).

## Architecture: one file does almost everything

`index.html` is the entire site — markup, page logic, and SEO metadata all live in it (~637 lines). There is no JS module split. Key inline subsystems in the `<script>` at the bottom:

- **Theme toggle** — adds/removes `dark` class on `<html>`, persists via `localStorage.theme`. Initial value also honors `prefers-color-scheme`.
- **Font toggle** — adds/removes `font-m5x7` class on `<body>`, persists via `localStorage.font`. When `m5x7` is active, `updateTitleFonts()` walks every `h1/h2/h3`, wraps each letter in a `<span>`, and applies the vibrate animation defined in `css/styles.css`.
- **Project details expand/collapse** — each `<article class="project-card">` has `onclick="toggleDetails('<id>')"`; the matching `<div id="<id>" class="project-details">` toggles its `active` class. The `id` is the contract — must be unique per card.
- **Image viewer modal** (`#imageViewer`) — reads images from a `.project-gallery[data-project="<id>"]` container; supports keyboard nav (←/→/Esc) and click-outside-to-close. Currently most project cards do not use the gallery; the modal code is present and ready for cards that include one.
- **"Just Me" photo modal** (`#meModal`) — separate, simpler modal wired by an IIFE at the bottom of the script.
- **Years of experience** — computed at `DOMContentLoaded` from `data-start-date` on each `.technology-level`. Do not hardcode year numbers in the markup; set `data-start-date` instead.

## Adding a project card

Each project is an `<article class="project-card">` block inside `.projects-grid` in `index.html`. To add one, duplicate an existing card and update:

1. `onclick="toggleDetails('<unique_id>')"` on the outer `<article>`.
2. The thumbnail `<img>` in `images/projects/<file>.jpg` — keep the 460×215 Steam capsule dimensions (or update the `width`/`height` attributes), and keep `loading="lazy"`.
3. The inner `<div id="<unique_id>" class="project-details">` — `id` must match step 1.
4. Steam link (with `rel="noopener noreferrer"`), title (`<h3 class="project-title">`), description, and the YouTube embed `src` inside `.project-video` (keep `loading="lazy"` on the iframe).
5. Add a matching entry to the `ItemList` JSON-LD (`VideoGame`) in `<head>` so the game is indexable by search engines.

The `id` is the only thing tying the click handler to the details panel, so a typo there silently breaks the expand behavior.

## SEO surface

The `<head>` of `index.html` contains the canonical SEO surface: meta description/keywords, favicon + `theme-color`, Open Graph + Twitter Card (which reference `images/social_card.png`, a 1200×630 social image), and two JSON-LD blocks: a `Person` schema and an `ItemList` of shipped games (`VideoGame`). `robots.txt` and `sitemap.xml` at the repo root are also part of the SEO surface — keep `sitemap.xml`'s `<lastmod>` reasonable when making content changes. `google43b4c03d1c8cf896.html` is a Google Search Console verification file; do not delete or rename it.

## Image assets

- `images/projects/` — project card thumbnails (one `.jpg` per shipped title, 460×215 Steam capsule format).
- `images/skills/` — technology icons; each has a `_light` and `_dark` variant, swapped via Tailwind's `dark:` classes (`block dark:hidden` / `hidden dark:block`). When adding a new skill, ship both variants.
- `images/me/` — avatar (also used as favicon / `apple-touch-icon`) and "Just Me" photo.
- `images/social_card.png` — 1200×630 image referenced by `og:image` and `twitter:image`. If you change it, keep the dimensions or update the `og:image:width`/`og:image:height` meta tags to match.
