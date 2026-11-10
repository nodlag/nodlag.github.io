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
- **Custom CSS** lives in `css/styles.css` (CSS variables for light/dark theme, project card styles, vibrate animation). Each selector is defined once; dark-mode overrides are grouped at the bottom.
- **Fonts are self-hosted** in `fonts/`: `m5x7.woff2` (pixel body font, with `m5x7.ttf` as fallback source) and `BebasNeue-Regular.woff2` (titles, Latin subset of the OFL font). Their `@font-face` rules are at the top of `css/styles.css`, and both `.woff2` files are preloaded from `<head>` so text does not shift when they arrive. Do not go back to Google Fonts or drop the preloads — that combination produced a 0.44 CLS on slow connections. If a font file is renamed, update the `@font-face` rule, the preload in `index.html` and the copies in `404.html`.

## Deploy

Pushing to `main` deploys to GitHub Pages — there is no staging environment. Treat every commit to `main` as production.

GitHub Pages builds the repo with Jekyll, which publishes every file it finds (and renders `.md` files to HTML). `_config.yml` exists only to `exclude` development files (`CLAUDE.md`, the Tailwind config and input). Add any new dev-only file at the repo root to that list.

## Local preview

Open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`) from the repo root. Use a local server rather than `file://` when testing anything that may be affected by CORS (fonts, iframes) — YouTube embeds in particular refuse to play without a referrer.

## Architecture: one file does almost everything

`index.html` is the entire site — markup, page logic, and SEO metadata all live in it. There is no JS module split. Key inline subsystems in the `<script>` at the bottom:

- **Preferences** — `readPref()` / `savePref()` wrap `localStorage` in `try/catch`. Always go through them: direct `localStorage` access throws when cookies are blocked, and an uncaught exception there stops the rest of the script.
- **Theme toggle** — adds/removes `dark` class on `<html>`, persisted as the `theme` preference. Initial value also honors `prefers-color-scheme`.
- **Font toggle** — adds/removes `font-m5x7` class on `<body>`, persisted as the `font` preference. When `m5x7` is active, `updateTitleFonts()` walks every `h1/h2/h3` and `wrapLettersInSpans()` rebuilds it as one `.title-word` span per word containing one `.title-letter` span per letter (the vibrate animation in `css/styles.css` targets the letters). Words are `white-space: nowrap`, so titles only wrap between words. The original text is kept in `data-title-text` to restore it. Because the function rewrites the heading from its text, keep `h1/h2/h3` free of child elements.
- **Project cards** — each `<article class="project-card">` has a `.project-header` (capsule image, `.project-title`, `.project-role`) and a `<button class="project-toggle">` stretched over that header. Clicking it toggles `is-open` on the article and `aria-expanded` on the button; CSS expands `.project-details` by animating `grid-template-rows` (no fixed `max-height`) and hides the collapsed panel with `visibility` so its links stay out of the tab order. Only the header toggles — clicks inside the details do nothing.
- **Trailers** — `.project-video` is an empty box with a `data-video-id`. The script creates the `youtube-nocookie.com` iframe when the card opens and removes it when the card closes (which also stops playback). Do not put `<iframe>` tags back in the markup: seven players used to load on every visit (about 3 MB plus ad requests) even with `loading="lazy"`.
- **"Just Me" photo modal** (`#meModal`) — wired by an IIFE at the bottom of the script; closes with the ✕ button, a click on the backdrop, or Escape. Its `<img>` is `loading="lazy"` so the photo is not downloaded while the modal is closed.
- **Years of experience** — computed from `data-start-date` on each `.technology-level` and written as `+ N Years`. The markup holds a `Since <year>` fallback (shown without JavaScript); keep that year equal to `data-start-date`, and never hardcode the number of years.

## Adding a project card

Each project is an `<article class="project-card">` block inside `.projects-grid` in `index.html`. To add one, duplicate an existing card and update:

1. The thumbnail `<img>` in `images/projects/<file>.jpg` — keep the 460×215 Steam capsule dimensions (or update the `width`/`height` attributes), and keep `loading="lazy"`.
2. `<h3 class="project-title">` (plain text only) and `<p class="project-role">`, a few words on what Javier did on that game. Both are always visible under the capsule.
3. A unique `id` on `<div class="project-details">`, and the same value in `aria-controls` on the `.project-toggle` button. The script finds the panel through the card, so a mismatch does not break the click, but keep them in sync for assistive tech.
4. Steam link (with `rel="noopener noreferrer"`) and the two description paragraphs.
5. `data-video-id` on `.project-video` with the YouTube video id (the part after `watch?v=`). Remove the `.project-video` div entirely if the game has no trailer.
6. Add a matching entry to the `ItemList` JSON-LD (`VideoGame`) in `<head>` so the game is indexable by search engines.

The button's accessible name and the trailer's `title` are generated from `.project-title`, so there is nothing else to label.

## SEO surface

The `<head>` of `index.html` contains the canonical SEO surface: meta description/keywords, favicon + `theme-color`, Open Graph + Twitter Card (which reference `images/social_card.png`, a 1200×630 social image), and two JSON-LD blocks: a `Person` schema and an `ItemList` of shipped games (`VideoGame`). `robots.txt` and `sitemap.xml` at the repo root are also part of the SEO surface — keep `sitemap.xml`'s `<lastmod>` reasonable when making content changes. `google43b4c03d1c8cf896.html` is a Google Search Console verification file; do not delete or rename it.

`404.html` is the custom not-found page GitHub Pages serves for any missing path. It is self-contained (inline styles, `noindex`) and uses absolute URLs because it can be served at any depth.

## Image assets

- `images/projects/` — project card thumbnails (one `.jpg` per shipped title, 460×215 Steam capsule format).
- `images/skills/` — technology icons; each has a `_light` and `_dark` variant, swapped via Tailwind's `dark:` classes (`block dark:hidden` / `hidden dark:block`). When adding a new skill, ship both variants.
- `images/me/` — avatar (also used as favicon / `apple-touch-icon`) and "Just Me" photo.
- `images/social_card.png` — 1200×630 image referenced by `og:image` and `twitter:image`. If you change it, keep the dimensions or update the `og:image:width`/`og:image:height` meta tags to match.
