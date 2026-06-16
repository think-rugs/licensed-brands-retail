# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single Next.js site (App Router, Next 15, React 19) that presents Think Rugs'
licensed-brand rug collections: a landing page at `/` plus one brochure route per
brand. It replaces an older suite of standalone single-file HTML brochures. The
build is a **full static export** (`output: 'export'` in `next.config.mjs`), so
`out/` deploys to any static host with no Node in production. `trailingSlash: true`
and `images.unoptimized: true` are required for the export and should not be changed
casually.

## Commands

Node is managed by **fnm**, pinned to the version in `.nvmrc` (24.16.0). If `node`/`npm`
are not on PATH, activate it first:

```bash
export PATH="$HOME/.fnm:$PATH" && eval "$(fnm env)"
```

```bash
npm install
npm run dev        # develop at http://localhost:3000
npm run build      # static export to out/
npm run preview    # serve the export locally (npx serve out)
```

There is **no test suite, linter, or typecheck** configured. The Python data/asset
scripts under `scripts/` need `python3` with `openpyxl` / `Pillow` (see each script
header). Verify changes by running `npm run dev` and exercising the affected route.

## Architecture

The whole site is data-driven from three JSON files in `data/`. Adding or changing a
brand is almost always a data edit, not a code edit.

- **`data/brand_data.json`** — the product catalogue, keyed by brand key (e.g. `Scion`,
  `LLB`, `Catherine Lansfield`). Each brand has `designs[]`; each design has
  `colourways[]`; each colourway has `sizes[]`. Optional `design.group` (e.g. "Woven" /
  "Washable") enables collection grouping in the UI automatically when a brand has two
  or more distinct group values.
- **`data/themes.json`** — per-brand identity: colours, fonts, copy (`BLURB`, `FOOTER`),
  logo keys, and tile overrides. These values flow into the brochure as **CSS custom
  properties** set on the `.brochure` wrapper (see `vars` in `BrandBrochure.jsx`), so
  all brands share one stylesheet (`app/brochure.css`). `SUB`/`RAIL_SUB` override the
  default "Washable Rug Collection" copy.
- **`data/img_manifest.json`** — which colourways (`{CODE}`) have `cut`/`life`/`detail`
  shots. Written by the asset scripts, read at build time to decide image availability.

### Key files

- **`lib/catalogue.js`** — the data access layer. Maps brand keys → routes (`ROUTES`),
  computes live landing-page stats and cards, and `getBrand()` prepares catalogue data
  (sorts runners to the bottom of size lists, merges image paths from the manifest).
  `getDownload()` gates the Excel link on a file existing in `public/downloads/` — a new
  brand's download stays hidden until its filename is added here. `getSelKey()` defines
  the localStorage selection key.
- **`components/BrandBrochure.jsx`** — the entire brochure as one `'use client'`
  component (~570 lines): search, colour/collection filters, Photographed-only / trade-price
  / selection toggles, selection persisted to `localStorage` per brand
  (`thinkrugs_{brand}_selection_2025`), CSV export, designs nav with scrollspy, product
  popup with gallery (swipe + keyboard), and a near-fullscreen lightbox.
- **`app/[brand]/page.js`** — each brand route is a thin server component: pick a brand
  `KEY`, then render `<BrandBrochure>` with `getTheme/getBrand/getDownload/getSelKey`.
  Routes use marketing slugs (`/scion-living`, `/clarke-and-clarke`, `/house-llewelyn-bowen`)
  that differ from brand keys.
- **`app/page.js`** — landing page; lays brands on a count-aware grid so the full set
  fits one viewport. `app/layout.js` loads all substitute fonts in one request.

### Brand keys vs routes vs slugs

Three identifiers per brand can differ and must stay consistent: the **brand key** (the
`data`/`themes` JSON key, e.g. `Clarke & Clarke`), the **route** (in `ROUTES`, e.g.
`/clarke-and-clarke`), and the **logo_key** (filename stem under `public/images/logos/`).
When wiring a new brand, update `themes.json`, `brand_data.json`, `ROUTES` and the
brand's `app/<slug>/page.js`.

## Data & asset pipeline (`scripts/`)

These regenerate the JSON and static assets; always `npm run build` after running one.

- `add_images.py <folder>` — processes incoming photography to standard sizes (cutouts
  690×920 white-padded, lifestyles 760 wide) into `public/images/products/` and updates
  `img_manifest.json`. Safe to re-run (`--force` to overwrite); accepts several filename
  conventions; reports unmatched/corrupt files rather than guessing.
- `export_assets.py <legacy_project>` — legacy path: decodes the old image cache and
  refreshes the manifest.
- `extract_llb.py <workbook.xlsx>` — rebuilds **only** the `LLB` block in `brand_data.json`
  from the product workbook (other brands untouched).
- `extract_cl.py <workbook.xlsx>` — splits one supplied workbook into both
  `Catherine Lansfield` and `CL Kids` blocks (keys off the `"CL Kids - "` Description
  prefix). Normalisation rules are documented at the top of the script.
- `llb_logo.py` — renders the LLB cover/rail logos from the source PDF.
- `single_file_preview.py <BRAND_KEY>` — inlines theme/logo/Excel/photos into one
  offline HTML file (the emailable copy). Vanilla-JS port of `BrandBrochure.jsx`.

## Conventions to preserve

- **Copy style: no em dashes or long hyphens** anywhere — only commas, colons,
  parentheses, full stops. This applies to content, README, and these notes.
- The brands maintained by `extract_*` scripts carry **drafted copy** (titles, descriptions,
  design intros) where official copy is missing; the scripts report this on every run.
  Don't treat drafted copy as final.
- Empty/`null` spec fields and missing downloads are hidden by design — a brand can be
  wired in with an empty catalogue (landing tile shows "Collection to follow").
- All artwork requires licensor marketing approval before publication.
