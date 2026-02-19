# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- **Local development server**: `hugo server` — builds and serves the site at http://localhost:1313 with live reload
- **Build for production**: `hugo --minify` — outputs static files to `public/`
- **Deploy**: Push to `main` branch; GitHub Actions (`.github/workflows/hugo.yml`) automatically builds with Hugo 0.148.2 extended and deploys to GitHub Pages

## Architecture

This is a **Hugo static site** using the **PaperMod theme** (located in `themes/`), customized for an academic personal website. The site is published at https://www.horaciomorenom.me/.

### Key configuration

`config.yml` is the single configuration file. It controls:
- Site metadata and `profileMode` (homepage with photo, subtitle, and buttons)
- Navigation menu (CV, Projects, Thoughts, Keywords)
- Social icons (CV, Email, YouTube, GitHub, Substack, Twitter)
- Math rendering is enabled globally (`math: true`)
- Theme is locked to `light` with no toggle (`disableThemeToggle: true`)

### Content structure

- `content/projects/` — research project pages, each in its own subdirectory with an `index.md` plus associated PDFs, images, and MP4 videos
- `content/thoughts/` — blog-style posts
- `content/archive.md`, `content/tags/`, `content/location.md`, `content/officehours.md` — auxiliary pages
- `static/` — global static assets: `cv.pdf`, `picture.jpg`, favicons

### Custom layouts

Layouts in `layouts/` override the PaperMod theme defaults:
- `layouts/_default/` — overrides for `baseof.html`, `single.html`, `list.html`, `search.html`, `archives.html`, `terms.html`
- `layouts/partials/` — overrides for header, footer, analytics, math, TOC, author, and other partials
- `layouts/_default/_markup/render-link.html` — custom link rendering

### Adding a new project

Create a new directory under `content/projects/` with an `index.md` using the front matter pattern from `archetypes/paper.md`:
```yaml
title: "..."
date: YYYY-MM-DD
tags: ["keyword1", "keyword2"]
author: ["Author Name"]
description: "..."
summary: "..."
cover:
    image: "filename.png"
    relative: false
```

Place all associated files (PDFs, images, videos) in the same directory and reference them with relative paths.

### Themes and assets

- PaperMod theme is vendored in `_vendor/` (used by Hugo modules) and in `themes/`
- Custom CSS overrides are in `assets/css/` (organized into `common/` and `core/` subdirectories)
- The built site in `public/` is not committed — GitHub Actions builds it on each push to `main`
