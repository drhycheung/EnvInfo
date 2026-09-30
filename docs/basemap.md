# Basemap notes: why Esri, and how to verify a tile layer

Companion note to the [main README](../README.md). Everything here concerns one line of code in
`index.html` — the `L.tileLayer(...)` call in `initMap()` — and the two ways it can fail silently.

**Last verified: 30 September 2026**, against the live Esri tile service, a local `file://` load, and
the deployed page at <https://drhycheung.github.io/EnvInfo/>. External tile services change their
terms and authentication without notice, so treat anything older than the date above as suspect —
see the CARTO note below for a live example of that happening.

## Contents

1. [Why the basemap is Esri, not OpenStreetMap](#1-why-the-basemap-is-esri-not-openstreetmap)
2. [Verifying a tile layer before you trust it](#2-verifying-a-tile-layer-before-you-trust-it)

---

## 1. Why the basemap is Esri, not OpenStreetMap

This is the single most important gotcha in the whole project, and it is worth a full section
because it is invisible until you double-click the file.

**The symptom.** The dashboard works perfectly when served over HTTP. Double-click `index.html`
(`file://` protocol) and the map area fills with grey squares — but the browser console is clean,
the network tab shows `HTTP 200` for every tile, and no error is ever logged.

**The cause.** OpenStreetMap's official tile servers (`{s}.tile.openstreetmap.org`) enforce a
**Referer requirement**: every tile request must carry a `Referer` header. A `file://` page cannot
send a `Referer` at all, so OSM responds with status `200` but substitutes a *"Access blocked: App is
not following the tile usage policy"* refusal image — a constant **6,987-byte** PNG in place of each
tile. The status code is what makes this so confusing: the browser considers the request successful.

**Fixes that do *not* work** (all verified, do not retry them):

| Attempted fix | Why it fails |
|---|---|
| `<meta name="referrer" content="unsafe-url">` or `content="origin"` | The Referrer-Policy header is ignored for `file://` origins; still no `Referer` is sent |
| CARTO basemaps (`basemaps.cartocdn.com`) | Now require an API key; without one every location/zoom returns the same constant **2,513-byte** blank placeholder |
| Changing the `{s}` subdomain (`a`/`b`/`c`) | Subdomains are not the problem — the policy check is server-side on every request |

The CARTO entry is worth dwelling on for students: CARTO basemaps were a genuine, widely recommended
`file://` workaround and keyless use worked for years. It stopped working quietly, with no deprecation
notice — the map simply turned blank. This is the general lesson: **a tile layer that renders today
may not render next year**, and the failure will not announce itself.

**The fix used.** Esri's ArcGIS Online basemaps, which do not require a `Referer` and need no API key:

```js
L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/{z}/{y}/{x}', {
  attribution: 'Tiles &copy; <a href="https://www.esri.com/">Esri</a> &mdash; Source: Esri, HERE, Garmin, &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
  maxZoom: 16
}).addTo(map);
```

Three details that are easy to get wrong:

1. **Path order is `{z}/{y}/{x}`, not `{z}/{x}/{y}`.** This is Esri's convention and the reverse of
   OSM's. Swap them and every URL 404s into a blank grey map — which looks identical to the
   original bug, so it is easy to misdiagnose.
2. **`maxZoom` must be `16`, not `18` or `19`.** `Canvas/World_Light_Gray_Base` has no data beyond
   zoom 16. Verified: a zoom-17 tile returns a constant **2,521-byte** blank image. Leaving
   `maxZoom: 19` gives a map that works perfectly at city scale and then silently turns to blank
   tiles the moment a user zooms in one more level.
3. **Attribution is mandatory** — Esri, HERE, Garmin and OpenStreetMap contributors. Keep the string
   intact (rendered via Leaflet's `attribution` option and repeated in the page footer).

Other Esri Canvas services worth knowing, all with the same `{z}/{y}/{x}` order and `maxZoom: 16`:
`World_Dark_Gray_Base` (dark — deliberately avoided here so the green-to-purple marker colours stay
the most salient thing on screen), `World_Topo_Map`, `World_Imagery` (satellite), `World_Street_Map`,
`World_Boundaries_and_Places` (a transparent overlay that can be stacked *on top of* one of the base
layers above).

---

## 2. Verifying a tile layer before you trust it

Because the failure mode returns `200`, "the map looks fine" is not a test. Check the bytes:

```js
// run in the browser console on the loaded page
const img = document.querySelector('img.leaflet-tile');
const buf = await (await fetch(img.src)).arrayBuffer();
console.log(img.src, buf.byteLength, img.naturalWidth);   // width must be 256
```

Reference values for the tile sizes you will see, which make silent substitutions easy to spot:

| Byte length | Meaning |
|---|---|
| ~**6,987** | OSM policy refusal image — your `file://` Referer problem |
| ~**2,513** / ~**2,521** | Blank placeholder — CARTO without an API key, or an Esri tile requested past `maxZoom: 16` |
| **~2,800–15,000** | Genuine tile. Small values are legitimate: a uniformly-coloured sea or countryside tile can compress to a couple of KB |

To distinguish a genuine uniform tile from a blank placeholder, decode it and count distinct pixel
values — a real tile over empty terrain is a single opaque colour, a placeholder is typically
transparent or pure white:

```js
const im = new Image(); im.crossOrigin = 'anonymous'; im.src = img.src; await im.decode();
const c = document.createElement('canvas'); c.width = c.height = 256;
c.getContext('2d').drawImage(im, 0, 0);
const d = c.getContext('2d').getImageData(0, 0, 256, 256).data;
new Set([...Array(d.length / 4)].map((_, i) => d.slice(i * 4, i * 4 + 4).join(','))).size;
```

A one-line `curl` check works too, and needs no browser:

```bash
curl -s -o tile.png -w "%{http_code} %{size_download}\n" \
  "https://server.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/13/3575/6693"
# 200 10827   <- status 200 and a realistic byte count
```

Compare `size_download` against the table above. A `200` on its own proves nothing.

---

Back to the [main README](../README.md) ·
Reproduce the project yourself: [vibe-coding guide](vibe-coding.md)
