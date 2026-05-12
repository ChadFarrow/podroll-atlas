# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Podroll Atlas — a single static `index.html` that renders ~28,000 podcasts as a navigable canvas network, mined from the Podcasting 2.0 `<podcast:podroll>` tag dataset published by Podcast Index. Live at [atlas.rss.io](https://atlas.rss.io/) (GitHub Pages, CNAME in repo root).

**There is no build step, no backend, no package manager, no test suite.** Everything — markup, CSS, JS, state — lives inline in `index.html`. D3 v7 is the only runtime dependency, loaded from `d3js.org` via `<script>` tag.

## Running locally

Open `index.html` in a browser. That is the entire workflow. For features that need a real origin (e.g. fetching from `public.podcastindex.org` without CORS surprises), serve the directory over HTTP instead of `file://`, e.g. `python3 -m http.server 8000`.

There is nothing to install, build, lint, or test.

## Critical: gitignore covers all JSON

`.gitignore` excludes `*.json`. The dataset (`recommendations.json`, ~16 MB) is intentionally never committed — it is fetched at runtime from `public.podcastindex.org` and cached in IndexedDB. If you generate or download dataset snapshots locally for testing, do not force-commit them.

## High-level architecture

All of the following live inside the single `<script>` block in `index.html` (starts at line 251). Section banners (`// ----- ... -----`) mark the boundaries; use them to navigate.

### Data flow

1. **`loadData(force)`** — Tries IndexedDB (24h TTL) → remote `REMOTE_URL` → local `LOCAL_FALLBACK` (`recommendations.json` next to HTML) → stale IndexedDB. Anything that succeeds gets cached back into IndexedDB.
2. **`buildIndexes(data)`** — Builds three maps that the rest of the app reads from: `byFeedId`, `recommendsByFeed` (source → recommended[]), `recommendersByFeed` (recommended feedId → recommender sources[]).
3. **`render()`** — Branches on the global `mode` (`"global"` vs `"focus"`), produces `{nodes, links}` via `buildGlobalGraph()` or `buildFocusGraph(feedId)`, then lays them out and draws.

### Mode state machine

- **`mode = "global"`** — All ~28k podcasts. Layout is **phyllotaxis** (golden-angle spiral) sorted by popularity, then a short `d3.forceCollide` relaxation pass. No per-frame force simulation. Edges exist in memory (full `sources[]` of every row) but are only drawn on hover for the hovered node — drawing all of them would obscure the map.
- **`mode = "focus"`** — Triggered by clicking a node or by `#<feedId>` in the URL. Small graph (focal podcast + neighbors), laid out by `d3.forceSimulation`. **Capped at `FOCUS_MAX_NEIGHBORS = 200` per direction** so aggregator accounts (one source recommends ~6,000 podcasts) don't kill the canvas. Full neighbor list still appears in the side info panel.

`focusFeed()` / `exitFocus()` are the transitions. Deep-linking is handled by `parseHashFeedId()` at boot.

### Rendering pipeline

- **`<canvas>`, not SVG.** All drawing is manual in `draw()`. Pan/zoom transform is `d3-zoom`'s `transform`, applied as a single `ctx.translate/scale` per frame.
- **Viewport culling** is done inline in `draw()` — nodes/edges outside the visible rect are skipped.
- **Cover artwork** is only loaded once a node renders ≥ `COVER_SCREEN_R` (22px) on screen. Below that threshold nodes are colored dots. `getImage()` memoizes loaded `Image` objects; failed loads are cached as `null` so we don't retry.
- **Edges are batched** into a single `beginPath()` / `stroke()` per color group, not one stroke per edge.
- **FPS counter** in the sidebar runs off `frameCount` in the render loop.

### Hit testing

`pickNode(clientX, clientY)` uses `d3.quadtree` (`quadtree` global, rebuilt by `rebuildQuadtree()` after every layout/simulation tick). Without it, click-testing 28k nodes would be O(n) per click.

### Click vs drag disambiguation

`d3.zoom` calls `stopImmediatePropagation()` on mousedown, so separate `mousedown`/`mouseup` listeners don't fire. The code watches `d3.zoom`'s own `start`/`end` lifecycle and treats a gesture as a click only if no `mousemove`/`wheel`/`touchmove` fired in between (`gestureMoved` flag, around line 846). When changing zoom/click behavior, preserve this pattern.

### RSS enrichment (focus mode only)

A meaningful share of podcasts appear only as *recommenders* — the dataset has their `feedId`/`url`/`host` but no title or cover. When focus mode opens, `applyEnrichment()` fetches their RSS feeds directly from the browser (up to `ENRICH_CONCURRENCY = 6` in parallel), parses `<title>` and `<itunes:image>`, and patches the node + side panel.

CORS is expected to fail on a large fraction of these fetches. The code logs once, caches the miss, and leaves the node as a placeholder. **Do not add retry loops or surface errors to the user** — silent graceful degradation is intentional.

### Dataset shape & backward compatibility

Each row in `recommendations.json` is a *recommended* podcast and carries:
- Metadata (`feedId`, `title`, `image`, `host`, `url`, …)
- `popularity` — recommender count
- `sources: [{feedId, url, host}, …]` — **every** recommender (new format)
- `sourceFeedId` / `sourceFeedUrl` / `recommenderHost` — legacy single-example fields kept as a fallback

`buildIndexes()` prefers `sources[]` and falls back to the legacy triple. If you touch this code, preserve the fallback — older cached payloads in users' IndexedDB still ship the legacy shape until the 24h refresh hits.

## Deploying

This repo is the GitHub Pages source for `atlas.rss.io` (see `CNAME`). Pushing to `main` is the deploy. There is no CI pipeline.

## Conventions worth knowing

- **Desktop-only by design.** The interaction model assumes mouse + keyboard (drag, scroll-wheel zoom, hover, Esc). A mobile notice is shown but the experience is not optimized for touch — don't invest in touch parity unless the user explicitly asks.
- **No frameworks, no bundler.** Resist the temptation to introduce one. The "single static HTML file" property is a feature: zero install, instant deploy, easy to fork.
- **Comments in code are sparse and load-bearing.** Existing comments explain *why* (e.g. the d3.zoom click-detection note, the FOCUS_MAX_NEIGHBORS cap). Match that style — don't add narration of *what* the code does.
