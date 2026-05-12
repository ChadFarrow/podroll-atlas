# Podroll Atlas

An interactive map of podcasts recommending other podcasts, mined from the Podcasting 2.0 `<podcast:podroll>` RSS tag.

A single static `index.html` (no build step, no backend) running entirely in the browser.

> **Heads up: this is a proof of concept / prototype.** A weekend-style experiment to see what the public Podcast Index podroll dataset looks like as a navigable network. Rough edges, opinionated defaults, and missing polish are expected. Issues and pull requests welcome, but please don't treat it as a finished product.
>
> **Desktop only.** The interaction model assumes a mouse and a real keyboard (drag to pan, scroll wheel to zoom, hover to inspect, Esc to exit focus). It will technically open on a phone or tablet but is not optimized for touch input or small screens.

## What you're looking at

Each dot is a podcast. The bigger the dot, the more times other podcasts in the index recommend it. Color encodes the hosting company (rss.com, Buzzsprout, Megaphone, etc.). Click a podcast to focus on it and see what it recommends (cyan arrows) and who recommends it (pink arrow). Search a title and the camera pans and zooms to the match.

## The dataset

Source: [podcastindex.org/datasets](https://podcastindex.org/datasets), specifically the `recommendations.json` file published by the Podcast Index project. The dataset aggregates every `<podcast:podroll>` tag found across the RSS feeds they index and ranks recommended podcasts by mention count.

At the time of writing:
- ~28,000 recommended podcasts (rows)
- ~16 MB of JSON
- Refreshed periodically on the Podcast Index side

### About `<podcast:podroll>`

`<podcast:podroll>` is one of the Podcasting 2.0 namespace tags. It lets a podcaster list other shows they recommend, right inside their own RSS feed. Some apps (Fountain, Podverse, podStation, etc.) surface those recommendations to listeners. This dataset is the aggregated, indexed view of all those tags.

## Caching

The 16 MB file is fetched once and stored in your browser's **IndexedDB** for 24 hours. Subsequent reloads read straight from the local cache, so:
- You don't hammer the public Podcast Index endpoint.
- The app opens instantly after the first visit.
- A "Force refresh" button bypasses the cache when needed.

`localStorage` was a non-starter for this since most browsers cap it at 5 to 10 MB. IndexedDB has practical limits of tens to hundreds of MB.

## Performance

All rendering, layout, and search happen on your machine. There is no server. Layout and frame rate therefore depend on the hardware running the app. The sidebar shows a live FPS counter so you can see how your machine is doing.

A few of the choices that keep it tractable for 28,000 nodes:

- **Canvas, not SVG.** SVG creates one DOM element per node and edge, which collapses past a few thousand. Canvas draws everything to a single bitmap, with browser-managed GPU acceleration where available (and an automatic CPU fallback if not).
- **Phyllotaxis layout for the global view, not force simulation.** A golden-angle spiral places the most-recommended podcasts near the center and the long tail outward. Instant placement, no per-tick force pass. A short collision-relaxation pass spreads out overlaps. Force simulation is only used inside focus mode where there are a few dozen nodes at most.
- **Batched edge drawing.** All edges in the global view share a single `beginPath()` / `stroke()` call instead of one per edge.
- **Viewport culling.** Nodes and edges outside the visible viewport are skipped each frame.
- **Lazy cover loading.** Cover artwork is only requested and drawn for nodes that render at least 22 pixels wide on screen. Until then they are colored dots. So zooming all the way out costs no image fetches.
- **Focus cap.** A handful of accounts act as podroll aggregators (one source recommends ~6,000 podcasts). Focus mode renders the top 200 by popularity to keep the canvas responsive. The full list is still shown in the side panel.

## D3.js

The app uses [D3](https://d3js.org/) for three things:

1. **`d3-force`** for the focus-mode layout (small graphs, ~10 to 200 nodes).
2. **`d3-zoom`** to handle the pan and zoom interaction on the canvas.
3. **`d3-quadtree`** for click hit-testing across the 28,000-node soup. The quadtree makes "what dot did the user click on?" an `O(log n)` lookup instead of scanning every node.

No D3 selection or DOM-binding is used for rendering. The graph is drawn manually to a `<canvas>`. D3 is here for its math and interaction primitives, not its rendering pipeline.

## The big dataset limitation

The JSON has one row per *recommended* podcast. Each row carries:
- The recommended podcast's metadata (title, GUID, image, host).
- A `popularity` count of how many other podcasts recommend it.
- A single example `sourceFeedId` of one of those recommenders.

So if Huberman Lab is recommended 133 times, the dataset will say `popularity: 133` and store **one** example recommender. The other 132 are aggregated into the count but their identities are gone.

What this means visually:
- **Outgoing arrows are complete.** When you focus on a podcast, every podcast it recommends is in the graph (its outgoing podroll items each appear as their own row, so we can find them all).
- **Incoming arrows are sparse.** We can only ever draw one pink arrow per focused podcast, even when the count says 133. The graph is therefore strongly directional, with rich fan-out and almost no fan-in.

If the dataset shipped the full edge list (all `(source, target)` pairs instead of one example per target), this visualization would suddenly become much richer:
- Genuine clusters would emerge from force layout (since connected podcasts would actually pull together).
- You could see who specifically recommends a popular show, not just the count.
- "Mutual recommendations" and small influence cliques would become visible.
- Community detection (e.g. Louvain) would actually have something to bite on.

Until then, the Atlas is best read as: a popularity map of recommended podcasts, with a click-to-explore tool for each podcast's outgoing podroll.

## Running it

Open `index.html` in a browser. That is the entire instruction.

The first load downloads ~16 MB from `public.podcastindex.org` and writes it to IndexedDB. Subsequent loads use the cache for 24 hours.

If the remote fetch fails for any reason (CORS, network, the endpoint is down), the app falls back to a local `recommendations.json` next to the HTML file, then to whatever is still in IndexedDB (even if stale). So once you've loaded it once, you can use it offline.

## Files

```
index.html              The whole app.
recommendations.json    Optional local copy of the dataset, used as fallback.
README.md               This file.
```

## Credits

- Built by **Alberto Betella** ([betella.net](https://betella.net)). Full disclosure: this prototype was vibe-coded, meaning most of the implementation was produced through a back-and-forth conversation with an AI coding assistant rather than typed from scratch. The product direction, dataset framing, design decisions and review are mine; the keystrokes are largely the model's.
- Data: [Podcast Index](https://podcastindex.org/) and every podcaster who wrote a `<podcast:podroll>` tag.
- The `<podcast:podroll>` tag itself: [Podcasting 2.0 namespace](https://github.com/Podcastindex-org/podcast-namespace).
- Visualization library: [D3.js](https://d3js.org/).
