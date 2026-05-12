# Podroll Atlas

An interactive map of podcasts recommending other podcasts, mined from the Podcasting 2.0 `<podcast:podroll>` RSS tag.

A single static `index.html` (no build step, no backend) running entirely in the browser.

**Try it live:** [atlas.rss.io](https://atlas.rss.io/)

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

## Dataset structure

The JSON has one row per *recommended* podcast. Each row carries:
- The recommended podcast's metadata (title, GUID, image, host).
- A `popularity` count of how many other podcasts recommend it.
- A `sources` array listing **every** recommender (`feedId`, `url`, `host`) of that podcast.
- A legacy `sourceFeedId` / `sourceFeedUrl` / `recommenderHost` triple that mirrors `sources[0]`, kept for backward compatibility.

So if Huberman Lab is recommended 133 times, the row will contain `popularity: 133` and a `sources` array of all 133 recommenders. The Atlas uses that array directly, drawing one real pink incoming arrow per recommender rather than synthetic "phantom" arrows.

> **Heads up about earlier versions.** Before the `sources` array was added, the dataset only stored *one* example recommender per podcast. The Atlas still falls back to phantom incoming arrows for the difference if it encounters an older cached payload (cleared on every 24h refresh, or via the "Force refresh" button). Once you reload against the current dataset, the incoming side becomes fully populated.

Practically:
- **Outgoing arrows are complete.** Every podcast a focused show recommends is in the graph as a node.
- **Incoming arrows are complete too** (as of the latest dataset). Each recommender is a real node, connected by a real pink arrow.
- **Focus mode caps at 200 nodes per direction** to keep the canvas responsive for super-popular shows or aggregator accounts. The full list is always shown in the side panel.

## RSS enrichment for source-only podcasts

A meaningful chunk of podcasts in the dataset appear only as *recommenders* (sources), never as recommended podcasts. For these, the JSON gives us their `feedId`, `url`, and `host` but no title or cover art. They'd otherwise render as anonymous "Feed #123456" dots with a default cover.

When you enter focus mode and any neighbor (or the focused podcast itself) is one of these source-only feeds, the Atlas:

1. Fetches each unknown podcast's RSS feed in the background (up to 6 in parallel).
2. Parses the channel `<title>` and `<itunes:image>` out of the XML.
3. Patches the title text under the graph node, swaps in the real cover art, and updates the matching row in the right-hand info panel.
4. Caches the result for the session, so re-focusing on the same neighborhood is instant.

**CORS fallback.** Many podcast hosts (especially smaller ones, self-hosted feeds, and some big platforms) don't send the `Access-Control-Allow-Origin` header on their RSS endpoint. Those fetches fail with a CORS error before the browser even shows the body to JavaScript. The Atlas handles this gracefully: the failure is logged once, the miss is cached so we don't retry, and the node stays as a placeholder with the default cover. No retry loop, no broken state, no user-visible error. The graph degrades smoothly.

In practice, hosts that tend to cooperate (RSS.com, Buzzsprout, Spreaker, parts of Anchor/Spotify, Megaphone, Captivate, Transistor, etc.) enrich successfully. The rest stay as anonymous nodes.

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

## License

Released under [**Creative Commons Attribution 4.0 (CC BY 4.0)**](https://creativecommons.org/licenses/by/4.0/). You're free to use, adapt, remix and redistribute, including commercially, as long as you give appropriate credit and link back to this repository.

See the [LICENSE](LICENSE) file for the full text.

## Credits

- Built by [**Alberto Betella**](https://betella.net). Disclosure: coded with heavy AI assistance. Architecture, design and decisions mine, most of the code AI-generated.
- Data: [Podcast Index](https://podcastindex.org/) and every podcaster who wrote a `<podcast:podroll>` tag.
- The `<podcast:podroll>` tag itself: [Podcasting 2.0 namespace](https://github.com/Podcastindex-org/podcast-namespace).
- Visualization library: [D3.js](https://d3js.org/).
