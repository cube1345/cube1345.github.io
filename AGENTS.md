# Repository Guidelines

## Project Structure & Module Organization
This repository is a Hexo blog. Core site configuration lives in [`_config.yml`](/home/cube/WorkSpace/FrontEnd/cubeblog/_config.yml), with content under [`source/`](/home/cube/WorkSpace/FrontEnd/cubeblog/source), post/page templates under [`scaffolds/`](/home/cube/WorkSpace/FrontEnd/cubeblog/scaffolds), and the active theme in [`themes/matery/`](/home/cube/WorkSpace/FrontEnd/cubeblog/themes/matery). Generated output goes to `public/` after a build, and Hexo’s local cache/database is stored in [`db.json`](/home/cube/WorkSpace/FrontEnd/cubeblog/db.json). Keep authored content in `source/_posts/*.md`; do not hand-edit generated files in `public/`.

## Build, Test, and Development Commands
Install dependencies with `npm install`. Use `npm run server` to start the local Hexo server for previewing changes. Use `npm run build` to generate the static site into `public/`. Use `npm run clean` before rebuilding if generated output or cache becomes stale. `npm run deploy` exists, but deployment is not configured in `_config.yml`; set `deploy.type` before relying on it.

## Coding Style & Naming Conventions
Follow the repository’s existing conventions: Markdown posts with YAML front matter, YAML config files with 2-space indentation, and lowercase, hyphenated filenames for content such as `source/_posts/my-first-post.md`. Keep front matter minimal and explicit, for example:

```md
---
title: My First Post
date: 2026-03-17
tags:
  - hexo
---
```

When editing the theme, preserve the existing file types and structure in `themes/matery/` (`layout/*.pug`, `languages/*.yml`, theme assets). Avoid committing `node_modules/`.

## Testing Guidelines
There is no automated test suite configured in `package.json`. Treat `npm run build` as the required validation step, then verify the site locally with `npm run server`. For content changes, check permalink generation, front matter parsing, and asset references. For theme changes, confirm the affected page renders correctly in the browser.

## Commit & Pull Request Guidelines
The root checkout does not include `.git` history, so no repository-specific commit pattern can be inferred locally. Use short, imperative commit messages such as `Add about page content` or `Adjust archive layout spacing`. Keep each commit focused on one concern. Pull requests should include a concise summary, note any config or theme files changed, link related issues when applicable, and attach screenshots for visible theme or content layout changes.

## Configuration Notes
The site currently uses the `matery` theme and stores site metadata in `_config.yml`. Review `url`, `theme`, and `deploy` settings before publishing. Theme-specific behavior should be configured in `themes/matery/_config.yml`, not hard-coded into post content.
