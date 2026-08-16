# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Project memory for `aadityavardhan.com` — a personal site for long-form, interactive
machine-learning articles. Read this before making changes.

## What this project is

A static Astro site publishing technical ML articles. The distinguishing feature is that
each article embeds **interactive playgrounds**: canvas-based widgets where the reader
manipulates data and watches the algorithm respond. The prose and the widget are meant to
be inseparable — if a widget could be swapped for a static image without losing the point,
it isn't earning its place and should be cut.

Audience: readers who already know some linear algebra and want the actual derivations,
not a hand-wave. Keep the math real. Show the objective being maximized, not just the
result.

## Stack

- **Astro** (≥7, Node ≥22.12), static output. No SSR adapter, no server rendering.
- **MDX** (`@astrojs/mdx`) for articles, so components can be embedded mid-prose.
- **Vanilla JS + `<canvas>`** for widgets. No React, no D3, no charting library.
- **KaTeX** loaded via CDN (`cdn.jsdelivr.net/npm/katex@0.16`) in `Layout.astro`. Use `$...$` for inline and `$$...$$` for display math in MDX.
- **Cloudflare Workers** static assets for hosting. Deploys on push to `main`.

Deliberately zero runtime dependencies for the interactive pieces. All the linear algebra
these articles need is 2x2 — closed-form eigenvalues and a `1/(ad−bc)` inverse. Adding a
matrix library would be more bytes than the math.

## Commands

```bash
npm run dev          # local dev server on :4321
npm run build        # static build into dist/
npm run preview      # serve the built output
npm run astro check  # TypeScript type-check
```

To run the dev server without blocking the terminal, use `astro dev --background`; manage it with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Deployment — read this before touching config

The site deploys to Cloudflare via `npx wrangler deploy`, triggered by a push to `main`.

`wrangler.jsonc` at the repo root is **load-bearing and must not be deleted**:

```jsonc
{
  "name": "aadityavardhan-site",
  "compatibility_date": "2026-08-08",
  "assets": { "directory": "./dist" }
}
```

Three separate things depend on it:

1. **Its existence** stops wrangler's auto-configuration routine from running. Without it,
   wrangler detects Astro and runs `astro add cloudflare` inside the build container,
   installing the SSR adapter. That switches prerendering into the workerd runtime and
   pulls in KV session and Cloudflare Images bindings this site does not use.
2. **The pinned `compatibility_date`** prevents an off-by-one failure. Auto-config stamps
   the current date, which can be newer than the workerd binary bundled with wrangler,
   producing `This Worker requires compatibility date "X" but the newest date supported
   by this server binary is "X-1"`. Bump this date manually and only deliberately.
3. **`assets.directory`** tells wrangler to upload `dist/` as plain static files with no
   Worker script.

Do not add `@astrojs/cloudflare` to `astro.config.mjs`. Do not add an `adapter:` line.
If a build log shows `astro add cloudflare` or `adapter: @astrojs/cloudflare`, the config
file is missing or was overridden — fix that rather than working around the symptom.

## DNS and domain

Cloudflare is authoritative for the zone. Both `aadityavardhan.com` and
`www.aadityavardhan.com` are attached to the Worker, and their DNS records are of type
`Worker` — created automatically by Cloudflare, **not** hand-written CNAMEs. Do not create
or edit DNS records for these hostnames manually; attach and detach them from the Worker's
Domains & Routes screen and let Cloudflare manage the records.

Outstanding, low priority: a redirect rule sending `www` to the apex, and null-MX + SPF +
DMARC records to prevent mail spoofing.

## Current state

The LDA article (`src/content/blog/lda.mdx`) is the first real article and is in progress.
The starter scaffolding (`Welcome.astro`, `src/assets/`) has been removed. Active components
are `src/components/PCAScatter.astro` and `src/components/LDAClassBuilder.astro`.
`src/layouts/Layout.astro` is the base HTML shell with global CSS custom properties and
KaTeX stylesheet.

## Content conventions

Articles live in `src/content/blog/` as `.mdx` and are served at `/blog/{filename}`.
Widgets live in `src/components/`.

Required frontmatter (schema enforced in `src/content.config.ts`):

```yaml
title: string
description: string
pubDate: date          # coerced — YYYY-MM-DD works
draft: boolean         # optional, default false; draft: true hides from index and build
```

Structure articles as: build the reader's intuition with a manipulable widget first, then
give the derivation, then show where the method **fails**. The failure section is not an
afterthought — it's usually the most valuable part and should get real space.

Write in plain declarative prose. State the objective function explicitly. When introducing
a method, say what it optimizes before saying what it computes.

## Widget conventions

Every interactive component follows the same shape. Match it when adding new ones.

- Single self-contained `.astro` file: markup, scoped `<style>`, and a `<script>` block.
- Vanilla TypeScript in the script. No framework imports.
- Query by class from a root element, then `for (const el of document.querySelectorAll(...)) init(el)`
  so multiple instances on one page work independently.
- Read colors through CSS custom properties defined on the root element, via
  `getComputedStyle`. Never hardcode hex values in the drawing code — the widgets must
  work in both light and dark mode. Tokens defined in `Layout.astro`:
  `--color-bg`, `--color-surface`, `--color-border`, `--color-text`, `--color-text-muted`,
  `--color-data-1` (blue/steel), `--color-data-2` (orange/red), `--color-axis`,
  `--color-widget-bg`, `--color-widget-border`.
- Handle `devicePixelRatio` in canvas setup or output looks blurry on retina displays.
- Re-render on `resize` and on `prefers-color-scheme` change.
- Provide `aria-label` on every canvas describing what it depicts.
- Numeric readouts use `font-variant-numeric: tabular-nums` so values don't jitter.
- **No `localStorage` or `sessionStorage`.** State lives in JS variables for the session.

Visual language across all widgets: white or near-black surface, hairline `1px` borders,
`2px` border radius, uppercase letter-spaced small-caps labels, restrained palette. The
widgets should read as scientific instruments, not dashboards. Two data colors maximum per
widget, chosen to stay distinguishable in both color schemes.

## Article in progress: Linear Discriminant Analysis

First article. Structure, in order:

1. **Recall PCA** — center, scatter matrix, the constrained maximization
   `max wᵀSw s.t. wᵀw = 1`, Lagrangian giving `Sw = λw`, variance explained. Closed-form
   2D eigen solution. Interactive click-to-add scatter showing PC1/PC2 recompute live.
   Ends on: no class label appears anywhere in the derivation.
2. **LDA, two classes** — projected means as a first objective, why it's inadequate
   (ignores within-class spread), scatter matrices `S_W` and `S_B`, arriving at the Fisher
   criterion `J(w) = wᵀS_Bw / wᵀS_Ww`.
3. **Fisher's linear discriminant** — differentiate, generalized eigenvalue problem,
   `w* ∝ S_W⁻¹(μ₁ − μ₂)`. Direct comparison against the PCA direction.
4. **Interactive class builder** — reader adds class 1 points, switches, adds class 2
   points. Designed so multimodal class structure visibly breaks the single discriminant
   axis.
5. **Assumptions and postmortem** — the honest section. Why LDA underperformed on a real
   binary patient-identification task over claims data where PCA worked well: binary
   indicators, sparse and zero-inflated counts, skew, outliers, and above all **phenotype
   heterogeneity in the positive class**. Worked example: positive class splits into a
   high-cardiac/low-respiratory group and a low-cardiac/high-respiratory group; the class
   mean describes neither, and one linear axis cannot represent both routes to membership.
   Also cover the `C−1` component ceiling and coefficient-interpretation caveats
   (correlated features splitting or flipping apparent importance, scaling sensitivity,
   conditional rather than causal contribution).

Sections are being built and reviewed **one at a time**. Do not draft ahead of the section
currently under discussion.
