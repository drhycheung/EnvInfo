# Hong Kong Air Quality Interactive Dashboard (GIS Teaching Demo)

**Live demo (GitHub Pages): https://drhycheung.github.io/EnvInfo/**

![Hong Kong Air Quality Interactive Dashboard screenshot](docs/screenshot.png)

A single-file, front-end-only interactive dashboard for visualising Hong Kong's real-time air
quality across all **18 EPD monitoring stations** (15 general + 3 roadside). Built for classroom
demonstration in **Environmental Informatics** and **Smart City** courses.

File: `index.html` — no build step, no backend, no API key. Open it in a
browser and it works; drop it into a GitHub Pages repository and it is deployed.

---

## 1. What the dashboard does

| Feature | Implementation |
|---|---|
| Interactive map of all 18 stations | Leaflet.js (CDN) + Esri Canvas **Light** Gray Base tiles (works over both `file://` and `http(s)` — see [§3.1](#31-why-the-basemap-is-esri-not-openstreetmap)) |
| Markers coloured by pollutant level | Simple 5-band graded colour scale, switchable per variable |
| Click popups | Station name (EN/中文), type, address, selected readings with units/bands, data timestamps |
| Variable selection | Checkboxes for AQHI, NO₂, O₃, SO₂, PM2.5, PM10 — map popups, markers and the data table update in sync |
| Colour metric | Radio buttons choose which variable drives marker colours |
| Data table | All 18 stations × selected variables, colour-coded cells |
| Bilingual interface | Every piece of on-screen text is presented in both Traditional Chinese and English |
| Auto-refresh | Polls every **300 seconds**; preserves checkbox state and never resets the map view |
| Error handling | On-page banner if either data source fails; last successful data is retained |
| Attribution | EPD/data.gov.hk source statement, Open-Meteo CC BY 4.0 credit, CAMS acknowledgement, Esri + OSM/Leaflet credits |

## 2. Data sources (and why two of them)

A key lesson of this project is that **the "obvious" official API cannot be used from a browser**:

| Source | Data provided | CORS policy | Browser? |
|---|---|---|---|
| EPD AQHI JSON | Official AQHI + health risk + publish time, all 18 stations, hourly | Open (`*`) | ✅ |
| Open-Meteo air quality | NO₂, O₃, SO₂, PM2.5, PM10 (CAMS model) at any lat/lon; batchable | Open (`*`), keyless | ✅ |
| EPD pollutant XML | Official station readings, past 24 h | Locked to aqhi.gov.hk | ❌ |
| City Dashboard map feed (POST) | Official readings **plus coordinates** | None sent | ❌ |

Full endpoint URLs:

```text
✅ https://dashboard.data.gov.hk/api/aqhi-individual?format=json
✅ https://air-quality-api.open-meteo.com/v1/air-quality
❌ https://www.aqhi.gov.hk/epd/ddata/html/out/24pc_Eng.xml
❌ https://dashboard.data.gov.hk/dashboard/smart_environment/data/map   (POST only)
```

So the dashboard fuses:

1. **EPD AQHI JSON** → official AQHI + health risk per station (join key: exact English station name).
2. **Open-Meteo air-quality API** → modelled pollutant concentrations at each station's hardcoded
   coordinates (all 18 stations fetched in **one batched request** by comma-separating coordinates).
   These are CAMS model values, clearly labelled "模擬" in the UI — *not* analyser readings.
3. **Hardcoded station metadata** — the 18 station names must match the API strings exactly
   (`Central/Western`, `Southern`, … `Mong Kok`). Coordinates come from the City Dashboard's
   published station addresses and the July 2020 government press release for the two newest
   stations (Southern = Aberdeen Tennis & Squash Centre; North = Po Wing Road Sports Centre).

> Teaching point: this is real-world spatial data fusion — real-time observations joined to static
> geometry metadata by name as primary key — plus an honest discussion of measurement vs model data.

## 3. How to run

- **Locally**: double-click `index.html` (any modern browser), or serve it:
  `python3 -m http.server 8000` then visit `http://localhost:8000/index.html`.
  Both routes work identically — including the map, which is why the basemap is Esri rather than
  OpenStreetMap (see [§3.1](#31-why-the-basemap-is-esri-not-openstreetmap)). The two APIs are
  `CORS *` and keyless, so nothing needs a local server; the only `file://` casualty is the
  OSM tile `Referer` requirement.
- **GitHub Pages (your own deployment)**: push `index.html` to *your* GitHub repository, then enable
  Pages via **Settings → Pages → Deploy from a branch** (select the branch and `/ (root)`). Your
  dashboard will go live at `https://<your-username>.github.io/<repo-name>/` — replace the
  placeholders with your own GitHub username and repository name.
  (The live-demo link at the top of this README is the author's own deployment.)

Desktop browsers assumed (no mobile optimisation, by design).

### 3.1 Why the basemap is Esri, not OpenStreetMap

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

### 3.2 Verifying a tile layer before you trust it

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

## 4. How this was built with OpenCode

The dashboard was produced in one OpenCode session using an agentic verify-first workflow:

1. **Endpoint discovery & verification** — OpenCode searched data.gov.hk, then used `curl` to fetch
   each candidate API, inspect headers (`access-control-allow-origin`) and JSON/XML structure.
   This exposed the CORS wall on the official XML and found the CORS-open AQHI endpoint.
2. **Fallback sourcing** — when no keyless CORS-open source of official pollutant readings existed,
   OpenCode verified Open-Meteo (keyless, `CORS *`, batch mode) against live HK coordinates.
3. **Geocoding the stations** — official coordinates for 16 stations were recovered from the City
   Dashboard's ArcGIS map feed; the two 2020 stations were located via the gov.hk press release and
   geocoded through Nominatim/Overpass (OpenStreetMap).
4. **Code generation** — a single annotated HTML file (native ES6, Chinese comments per block).
5. **Browser testing** — OpenCode drove the page with Playwright: checked console errors, confirmed
   18 table rows × 6 variables rendered, opened a popup, toggled checkboxes (table columns and
   popup content updated), switched the colour metric (markers recoloured, legend updated).

Two bugs were caught by measurement rather than by looking at the screen, and both are now encoded
into the student prompt in §6:

- **A `file://`-only broken basemap.** The map was fine over HTTP and full of grey squares when the
  file was double-clicked — with zero console errors and `HTTP 200` on every tile. The giveaway was
  the constant 6,987-byte tile size: an OSM policy refusal image, not a map. See [§3.1](#31-why-the-basemap-is-esri-not-openstreetmap).
- **An overlapping legend.** AQHI rendered as `≤ 3 / 3 – 6 / 6 – 7 / 7 – 10`, so adjacent bands
  claimed the same boundary values. The classifier used `value <= steps[i]` while the legend printed
  an inclusive range, so the two disagreed at every threshold. Fixed by deriving legend rows from the
  same thresholds with an exclusive lower bound, and by asserting that no two legend intervals
  overlap and that every threshold value lands in the row that claims it.

Total elapsed time: roughly one working session, most of it spent on steps 1–3 — which is exactly
why those findings are baked into the student prompt below.

## 5. Design Thinking: from raw data to an intuitive dashboard

The dashboard is the output of one design-thinking loop applied to a real usability problem:
Hong Kong's air quality data exist, but they are not intuitive.

| Stage | This project's arc |
|---|---|
| **1. Empathise 同理心** | The user pain: pollutant tables buried on official sites, technical units (μg/m³), AQHI split from pollutant detail, sources scattered across pages, little geographic context. Non-experts cannot quickly answer "is the air bad near me right now?" |
| **2. Define 定義** | Problem statement: *air quality information is available but not intuitive — residents need a single view that makes monitoring effortless and comparison instant.* Design goal: one screen, minimal jargon, at-a-glance status |
| **3. Ideate 構思** | Options to make data intuitive: colour-coded map markers (read status without reading numbers), graded legends instead of raw thresholds, health-risk wording alongside indices, click-for-detail popups, a full-data table, bilingual labels. Converged design: interactive Leaflet map + control panel + summary table |
| **4. Prototype 原型** | The single-file dashboard itself: every ideation choice materialised — green-to-purple marker colours, popups on demand, checkboxes to reduce clutter, 300-second auto-refresh so the page "monitors" without user effort |
| **5. Test 測試** | Browser-automated checks plus real-user feedback; each fine-tuning round improved intuitiveness — wider panel, fully bilingual text, clearer table spacing, an explicit modelled-vs-measured disclaimer |

Intuition was treated as the measurable outcome: every design decision traces back to reducing
the time from "open page" to "understood the air".

**Benchmark against the official service**: EPD operates its own air quality website at
[www.aqhi.gov.hk](https://www.aqhi.gov.hk). Its pollutant figures are **more accurate** —
direct measurements from the monitoring stations rather than modelled values — and remain the
authoritative reference. This project complements rather than replaces it: the focus here is on
intuitive at-a-glance visualisation, with data provenance labelled honestly and users pointed
back to EPD for authoritative readings.

> [!TIP]
> In this sense, the project is **not unique** — official and third-party air quality dashboards
> already exist. That is by design: it is a teaching baseline, not a novel product. Students are
> encouraged to explore extensions of this project, or similar projects of their own — new data
> layers, analyses, audiences or services — so that what they build brings genuinely unique value
> to environmental management.

## 6. Student reproduction prompt

Give the prompt below to Gemini, OpenCode, Claude, ChatGPT or any coding agent. It encodes every
pitfall discovered above, so a working dashboard should come out first-pass:

```text
Build a complete, standalone, single-file HTML page (all CSS/JS inline, native ES6 only,
no frameworks) for a Hong Kong air quality dashboard, deployable on GitHub Pages.

DATA SOURCES — use exactly these, they are verified working:
1. AQHI (official EPD, CORS-open):
   https://dashboard.data.gov.hk/api/aqhi-individual?format=json
   Returns an array: {station:"Central/Western", aqhi:2, health_risk:"Low",
   publish_date:"2026-08-21T09:30:00"} for 18 stations.
2. Pollutant concentrations NO2/O3/SO2/PM2.5/PM10 (Open-Meteo, keyless, CORS-open):
   https://air-quality-api.open-meteo.com/v1/air-quality
     ?latitude=<lat1,lat2,...>&longitude=<lon1,lon2,...>
     &current=pm10,pm2_5,nitrogen_dioxide,ozone,sulphur_dioxide&timezone=Asia/Hong_Kong
   Comma-separate ALL 18 station coordinates so ONE request returns an array covering
   every station (response order matches input order).
DO NOT use www.aqhi.gov.hk XML feeds or dashboard.data.gov.hk POST endpoints:
they lack CORS headers and will fail from a browser.

STATIONS: hardcode an array of all 18 stations with fields
{name, zh, type:'general'|'roadside', lat, lon}. The name string MUST exactly equal the
API's station field (it is the join key): Central/Western, Southern, Eastern, Kwun Tong,
Sham Shui Po, Kwai Chung, Tsuen Wan, Tseung Kwan O, Yuen Long, Tuen Mun, Tung Chung,
Tai Po, Sha Tin, North, Tap Mun, Causeway Bay, Central, Mong Kok.
Use plausible central coordinates for each district (general stations are rooftop sites,
roadside stations are at street level in Causeway Bay, Central and Mong Kok).

BASEMAP — use this tile layer verbatim, do NOT substitute OpenStreetMap or CARTO:
  L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/{z}/{y}/{x}', {
    attribution: 'Tiles &copy; <a href="https://www.esri.com/">Esri</a> &mdash; Source: Esri, HERE, Garmin, &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    maxZoom: 16
  }).addTo(map);
  WHY, and why the obvious alternatives fail — the page must work when a student
  double-clicks the HTML file (file:// protocol), not just when it is served over HTTP:
  - OSM's official tiles (tile.openstreetmap.org) impose a Referer requirement: every tile
    request must carry a Referer header. A file:// page cannot send one, so OSM returns
    HTTP 200 while substituting a constant ~6,987-byte "Access blocked: App is not following
    the tile usage policy" refusal image for every tile. The map fills with grey squares while
    the console stays clean and every request reports 200 — the status code is what makes this
    so confusing.
  - <meta name="referrer" content="unsafe-url"> / content="origin" do NOT help: the
    Referrer-Policy header is ignored for file:// origins.
  - CARTO basemaps (basemaps.cartocdn.com) now require an API key; without one every
    location and zoom returns the same constant ~2,513-byte blank placeholder.
  - Esri's ArcGIS Online Canvas basemaps impose no Referer requirement and need no API key,
    so the same file works under file:// and under http(s).
  THREE DETAILS THAT ARE EASY TO GET WRONG — check each one:
  1. The tile path order is {z}/{y}/{x} for Esri, NOT {z}/{x}/{y} like OSM. Reversing them
     yields a blank map that looks identical to the original bug.
  2. maxZoom MUST be 16 (not 18 or 19). Canvas/World_Light_Gray_Base has no data past zoom
     16 — a zoom-17 tile returns a constant ~2,521-byte blank image — so a map that looks fine
     at city scale silently goes blank when a user zooms in one more level.
  3. Keep the full attribution string (Esri, HERE, Garmin, OpenStreetMap contributors) in the
     tile layer's attribution option AND in the page footer; Esri requires visible credit.
  Use the LIGHT basemap as specified so the green-to-purple marker colours stay the most
  salient element on screen. If a dark map is ever wanted, the drop-in alternative is
  Canvas/World_Dark_Gray_Base (same {z}/{y}/{x} order, same maxZoom 16).

REQUIREMENTS:
- Leaflet.js via CDN with the basemap tile layer specified above; one circle-marker per station
  whose fill colour follows a simple 5-band graded scale of the currently selected colour metric
  (radio group, default AQHI); clicking a marker opens a popup showing the CHECKED variables with
  units, band label and both data timestamps.
- Band thresholds are a single source of truth, shared by the marker colours and the legend.
  Classify with `value <= steps[i]` (so steps[i] itself belongs to band i), and render each legend
  row from those same steps with an EXCLUSIVE lower bound: "≤ hi" for the first band,
  "> lo – hi" for the middle bands, "> lo" for the last. A legend that prints inclusive ranges
  ("≤ 3", "3 – 6", "6 – 7", "7 – 10") makes adjacent bands share a boundary value and is wrong:
  AQHI must read ≤3 / >3–6 / >6–7 / >7–10 / >10, i.e. 1-3 / 4-6 / 7 / 8-10 / >10 per EPD.
- Checkbox list (AQHI, NO2, O3, SO2, PM2.5, PM10, all checked initially): checking/unchecking
  updates markers' popups AND the data table simultaneously.
- Right sidebar: colour-metric radios, variable checkboxes, dynamic legend, and a data table
  listing all 18 stations x checked variables with colour-coded cells.
- Auto-poll both APIs every 300 seconds WITHOUT resetting checkbox state or the map view
  (update markers in place with setStyle/setPopupContent; never re-create the map).
- Show a red error banner if either request fails and keep the last good data.
- Label AQHI values as "EPD official" and pollutant values as "modelled (CAMS)".
- Include an attribution footer crediting: EPD via DATA.GOV.HK; Air quality data by
  Open-Meteo.com under CC BY 4.0 (underlying data Copernicus CAMS/ECMWF);
  Esri (Canvas World Light Gray Base, source Esri/HERE/Garmin) and © OpenStreetMap contributors;
  Leaflet.
- Add generous Chinese comments explaining each functional block (teaching use), including the
  Referer explanation and the {z}/{y}/{x} + maxZoom notes beside the tileLayer call, so students
  reading the code see WHY the basemap is not OpenStreetMap.

BEFORE YOU FINISH — verify the basemap by byte size, not by eye, because the failure mode
returns HTTP 200 and looks like success:
- Load the page over file:// and confirm document.referrer is "" (empty).
- Confirm every img.leaflet-tile loaded at naturalWidth 256 and came from server.arcgisonline.com.
- Fetch one tile and print its byte length: expect roughly 2,800-15,000 bytes. A value near 6,987
  means an OSM refusal image; near 2,513 or 2,521 means a blank placeholder (CARTO without a key,
  or an Esri tile requested beyond maxZoom 16).
- Zoom to levels 11, 13 and 16 and confirm the byte lengths differ per level and none is blank.
- Confirm the console has zero errors, that markers/popups/the data table still work, and that the
  zoom-in control disables itself at zoom 16.
```

## 7. Licences & attribution

- AQHI data: Environmental Protection Department, HKSAR Government, via [DATA.GOV.HK](https://data.gov.hk).
- Pollutant concentration layer: [Air quality data by Open-Meteo.com](https://open-meteo.com/),
  licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); underlying data from the
  Copernicus Atmosphere Monitoring Service (CAMS), ECMWF.
- Basemap: Tiles © [Esri](https://www.esri.com/) — *Canvas World Light Gray Base*; source Esri, HERE,
  Garmin, © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright).
  Map library: [Leaflet](https://leafletjs.com).

  Esri's ArcGIS Online basemaps are used instead of OSM's official tiles because they impose no
  `Referer` requirement, which is what allows the page to work when opened directly as a local file.
  OSM is still credited as a data source of the basemap. See [§3.1](#31-why-the-basemap-is-esri-not-openstreetmap).
