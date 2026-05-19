# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

HAMi GPU virtualization workshop — MkDocs Material site with structured experiment chapters teaching HAMi commercial edition features.

## Commands

```bash
# Activate Python environment
source .venv/bin/activate

# Local dev server (http://localhost:8000)
mkdocs serve

# Build static site to site/
mkdocs build

# Strict build (fails on warnings)
mkdocs build --strict
```

## Project Structure

- `mkdocs.yml` — Site config, navigation, theme settings
- `docs/` — All markdown content organized by chapter:
  - `installation/` — HAMi install + license
  - `basics/` — GPU node, sharing, card scheduling
  - `scheduling/` — Binpack/spread, priority
  - `advanced/` — Memory scaling, resource quota, analysis, override
  - `monitoring/` — HAMi metrics
  - `configuration/` — Global settings
- `sources/` — K8s YAML manifests referenced by experiments
- `screenshot/` — PNG images used in docs

## Editing Conventions

- Content in Chinese, technical terms in English
- Image paths from `docs/subdir/` use `../../screenshot/`
- Source YAML links from `docs/subdir/` use `../../sources/`
- Root-level docs (exam.md, faq.md) use `../screenshot/` and `../sources/`
- HAMi annotations reference: see `docs/installation/online-install.md` for full annotation list
