# Reproducing the dashboard with vibe coding

Companion guide to the [main README](../README.md). This is the teaching resource for the
lesson: the reasons for the design of the dashboard, how it was built with an AI coding tool,
and the complete prompt required to reproduce it.

**Last verified: 30 September 2026** using Gemini Canvas and OpenCode, and against the deployed
page at <https://drhycheung.github.io/EnvInfo/>.

## Contents

1. [Design thinking: from raw data to an intuitive dashboard](#1-design-thinking-from-raw-data-to-an-intuitive-dashboard)
2. [How the dashboard was built](#2-how-the-dashboard-was-built)
3. [The reproduction prompt](#3-the-reproduction-prompt) ← jump here if you just want to build it

---

## 1. Design thinking: from raw data to an intuitive dashboard

The dashboard is the result of one design-thinking cycle applied to a practical usability
problem: air quality data for Hong Kong are published, but they are not presented in a form
that is easy to use.

| Stage | This project's arc |
|---|---|
| **1. Empathise 同理心** | The difficulty faced by users: pollutant tables are placed within official websites, technical units (μg/m³) are used throughout, the AQHI is presented separately from the pollutant measurements, information is distributed across several pages, and there is little geographic context. A reader who is not a specialist cannot quickly answer the question "is the air quality poor near me at present?" |
| **2. Define 定義** | Problem statement: *air quality information is published but is not presented in an accessible form; residents require a single view in which monitoring requires no effort and comparison is immediate.* Design objective: a single screen, minimal technical vocabulary, and a status that can be read at a glance |
| **3. Ideate 構思** | Options considered for improving accessibility: colour-coded map markers, so that status can be read without reading numbers; graded colour scales instead of raw threshold values; health-risk wording alongside the numerical indices; popups that display detail on request; a complete data table; and bilingual labels. The design adopted: an interactive Leaflet map, a control panel, and a summary table |
| **4. Prototype 原型** | The single-file dashboard: each option was implemented, with markers coloured from green to purple, popups displayed on request, checkboxes that reduce visual clutter, and automatic refresh every 300 seconds so that the page monitors conditions without user effort |
| **5. Test 測試** | Automated browser checks and feedback from real users; each round of adjustment improved accessibility, through a wider panel, fully bilingual text, clearer spacing in the table, and an explicit statement distinguishing modelled from measured data |

Accessibility was treated as the measurable outcome: each design decision reduces the time
between opening the page and understanding the current air quality.

### Context: Monitor, Analyse, Control

This project is the **monitor** stage of environmental informatics: it collects and displays
data so that the present situation is visible. It does not include analysis or control, and it
makes no prediction.

The reason is worth stating, because it defines the boundary of the project. A monitoring
dashboard can report only what has already been measured. It can answer "what is the air
quality like right now?" but it cannot answer "will tomorrow evening exceed 150 µg/m³?", and it
therefore cannot support a decision about tomorrow. Those two stages are covered by a separate
project, [EnvML](https://github.com/drhycheung/EnvML), which predicts the concentration for an
hour that has not yet occurred.

The two projects are intended to be used together, and the boundary between them is the
teaching point: monitoring without prediction produces description but no action.

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

---

## 2. How the dashboard was built

The dashboard was produced in one session using an agentic verify-first workflow. The tool used was
OpenCode driven through Playwright; the same prompt in [Part 3](#3-the-reproduction-prompt) works in
Gemini Canvas, ChatGPT, Claude or any other coding agent.

1. **Endpoint discovery & verification** — OpenCode searched data.gov.hk, then used `curl` to fetch
   each candidate API, inspect headers (`access-control-allow-origin`) and JSON/XML structure.
   This exposed the CORS wall on the official XML and found the CORS-open AQHI endpoint.
2. **Fallback sourcing** — when no keyless CORS-open source of official pollutant readings existed,
   OpenCode verified Open-Meteo (keyless, `CORS *`, batch mode) against live HK coordinates.
3. **Geocoding the stations** — official coordinates for 16 stations were recovered from the City
   Dashboard's ArcGIS map feed; the two 2020 stations were located via the gov.hk press release and
   geocoded through Nominatim/Overpass (OpenStreetMap).
4. **Code generation** — a single annotated HTML file (native ES6, English comments per block).
5. **Browser testing** — OpenCode drove the page with Playwright: checked console errors, confirmed
   18 table rows × 6 variables rendered, opened a popup, toggled checkboxes (table columns and
   popup content updated), switched the colour metric (markers recoloured, legend updated).

Two bugs were caught by measurement rather than by looking at the screen, and both are now encoded
into the prompt in [Part 3](#3-the-reproduction-prompt):

- **A `file://`-only broken basemap.** The map was fine over HTTP and full of grey squares when the
  file was double-clicked — with zero console errors and `HTTP 200` on every tile. The giveaway was
  the constant 6,987-byte tile size: an OSM policy refusal image, not a map. See the
  [basemap notes](basemap.md).
- **An overlapping legend.** AQHI rendered as `≤ 3 / 3 – 6 / 6 – 7 / 7 – 10`, so adjacent bands
  claimed the same boundary values. The classifier used `value <= steps[i]` while the legend printed
  an inclusive range, so the two disagreed at every threshold. Fixed by deriving legend rows from the
  same thresholds with an exclusive lower bound, and by asserting that no two legend intervals
  overlap and that every threshold value lands in the row that claims it.

Total elapsed time: roughly one working session, most of it spent on steps 1–3 — which is exactly
why those findings are baked into the prompt below.

> [!IMPORTANT]
> These two bugs are the pedagogical heart of the lesson. Neither produced an error message; both
> produced a page that looked finished. The only reason they were caught is that someone measured
> the output instead of trusting it. An AI coding tool will readily produce a page containing either fault
> and describe it as working.

---

## 3. The reproduction prompt

Give the prompt below to Gemini, OpenCode, Claude, ChatGPT or any coding agent. It encodes every
pitfall discovered above, so a working dashboard should come out first-pass.

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
  WHY, and why the more apparent alternatives do not work — the page must function when a student
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
     at city scale becomes blank without any error when a user zooms in one more level.
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
- Add generous English comments explaining each functional block (teaching use), including the
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

---

Back to the [main README](../README.md) ·
Basemap and tile-layer notes: [basemap.md](basemap.md)
