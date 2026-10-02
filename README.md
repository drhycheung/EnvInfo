# Hong Kong Air Quality Interactive Dashboard (GIS Teaching Demo)

**Live demo (GitHub Pages): https://drhycheung.github.io/EnvInfo/**

![Hong Kong Air Quality Interactive Dashboard screenshot](docs/screenshot.png)

A single-file, front-end-only interactive dashboard for visualising Hong Kong's real-time air
quality across all **18 EPD monitoring stations** (15 general + 3 roadside). Built for classroom
demonstration in **Environmental Informatics** and **Smart City** courses.

File: `index.html` — no build step, no backend, no API key. Open it in a
browser and it works; drop it into a GitHub Pages repository and it is deployed.

---

## 1. Where this project fits: Monitor, Analyse, Control

Environmental informatics is usually taught as three activities: **monitor**, **analyse** and
**control**.

| Stage | What it means | Which project |
|---|---|---|
| **Monitor** | Collect and display data so that the current situation is visible | **This project** — a live dashboard of Hong Kong air quality |
| **Analyse** | Examine the data in order to explain patterns and relationships | [EnvML](https://github.com/drhycheung/EnvML) — a model relating weather to PM2.5 |
| **Control** | Act on the analysis, by deciding what to do next | [EnvML](https://github.com/drhycheung/EnvML) — the prediction, and the thresholds that trigger an action |

**This project is the monitor stage.** It answers the question *"what is the air quality like
right now, and where?"* It covers all 18 Environmental Protection Department stations and
refreshes automatically, so that the current situation is visible without any manual effort.

**Monitoring is necessary but not sufficient.** A dashboard displays what has already
happened. It cannot answer *"will tomorrow evening exceed 150 µg/m³?"*, and it therefore
cannot support a decision about tomorrow. Data that cannot be used for a decision has limited
value, however well it is displayed.

The **analyse** and **control** stages are covered by a separate project,
[EnvML](https://github.com/drhycheung/EnvML), which takes a data stream that could only be
watched and turns it into an estimate for an hour that has not yet occurred. The two projects
are designed to be used together: this project supplies the monitor stage, and EnvML supplies
the analyse and control stages.

Because this project deals only with observation and display, it deliberately makes no
prediction. Adding a forecast to a monitoring dashboard would change what the dashboard is,
and would require the analysis and accuracy reporting described in the EnvML README.

---

## 2. What the dashboard does

| Feature | Implementation |
|---|---|
| Interactive map of all 18 stations | Leaflet.js (CDN) + Esri Canvas **Light** Gray Base tiles — works over both `file://` and `http(s)`, see the [basemap notes](docs/basemap.md) |
| Markers coloured by pollutant level | Simple 5-band graded colour scale, switchable per variable |
| Click popups | Station name (EN/中文), type, address, selected readings with units/bands, data timestamps |
| Variable selection | Checkboxes for AQHI, NO₂, O₃, SO₂, PM2.5, PM10 — map popups, markers and the data table update in sync |
| Colour metric | Radio buttons select which variable determines the marker colours |
| Data table | All 18 stations × selected variables, colour-coded cells |
| Bilingual interface | Every piece of on-screen text is presented in both Traditional Chinese and English |
| Automatic refresh | Requests data every **300 seconds**; retains the checkbox settings and does not reset the map view |
| Error handling | Displays a message on the page if either data source fails, and retains the most recent successful data |
| Attribution | EPD/data.gov.hk source statement, Open-Meteo CC BY 4.0 credit, CAMS acknowledgement, Esri + OSM/Leaflet credits |

## 3. Data sources (and why two of them)

An important lesson of this project is that **the official API that appears to be the obvious choice cannot be used from a browser**:

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

The dashboard therefore combines three sources:

1. **EPD AQHI JSON** → official AQHI + health risk per station (join key: exact English station name).
2. **Open-Meteo air-quality API** → modelled pollutant concentrations at each station's hardcoded
   coordinates (all 18 stations fetched in **one batched request** by comma-separating coordinates).
   These are CAMS model values, clearly labelled "模擬" in the UI — *not* analyser readings.
3. **Hardcoded station metadata** — the 18 station names must match the API strings exactly
   (`Central/Western`, `Southern`, … `Mong Kok`). Coordinates come from the City Dashboard's
   published station addresses and the July 2020 government press release for the two newest
   stations (Southern = Aberdeen Tennis & Squash Centre; North = Po Wing Road Sports Centre).

> Teaching point: this is spatial data fusion as it is performed in practice — real-time
> observations joined to static location data, using the station name as the join key — together
> with a clear account of the difference between measured and modelled data.

## 4. How to run

- **Locally**: double-click `index.html` in any modern browser, or serve the directory with
  `python3 -m http.server 8000` and open `http://localhost:8000/index.html`. Both methods work
  identically, including the map, which is why the basemap is Esri rather than OpenStreetMap
  (see the [basemap notes](docs/basemap.md)). Both APIs return `CORS *` and require no key, so
  no local server is needed; the only element that does not work from `file://` is the OSM
  tile `Referer` requirement.
- **GitHub Pages (your own deployment)**: push `index.html` to *your* GitHub repository, then enable
  Pages via **Settings → Pages → Deploy from a branch** (select the branch and `/ (root)`). Your
  dashboard will go live at `https://<your-username>.github.io/<repo-name>/` — replace the
  placeholders with your own GitHub username and repository name.
  (The live-demo link at the top of this README is the author's own deployment.)

Desktop browsers are assumed. Mobile screens are not supported, by design.

## 5. Known limitations

These are stated explicitly because several of them follow from the constraints described in
section 3, and are not oversights:

| Limitation | Consequence |
|---|---|
| Pollutant values are **modelled** (CAMS via Open-Meteo), not measured | The dashboard is an aid to visualisation and is not an authoritative source. [EPD's own website](https://www.aqhi.gov.hk) publishes the official analyser readings |
| The station join key is the **exact English station name** | If EPD renames a station, that station disappears from both the map and the table without any error message |
| Coordinates are **hardcoded**, not fetched | New or relocated stations will not appear until the `STATIONS` array is updated by hand |
| The refresh interval is 300 s, but EPD publishes AQHI **hourly** | The map may lag the official figure by up to one publishing cycle; the publication time is shown in each popup so that the delay is visible |
| Basemap stops at **zoom 16** | `Canvas/World_Light_Gray_Base` has no data beyond it; street-level zoom on Esri's other services would be needed |
| Desktop only | No mobile or tablet layout; the sidebar is a fixed-width scrolling column |
| Four external dependencies | Leaflet (unpkg CDN), two live APIs and the Esri tile service must all be reachable; when offline, the tiles and the data fail together |
| The Esri attribution must remain visible | Removing or shortening the credit breaches Esri's terms of use — see the [basemap notes](docs/basemap.md) |

## 6. Documentation

| Document | What it covers |
|---|---|
| **[Basemap notes](docs/basemap.md)** | Why the basemap is Esri rather than OpenStreetMap; the `file://` `Referer` problem that returns `HTTP 200` with an image stating that access is refused; the three remedies that do *not* work; the `{z}/{y}/{x}` and `maxZoom: 16` problems; and how to verify a tile layer by file size rather than by visual inspection |
| **[Vibe-coding guide](docs/vibe-coding.md)** | The design-thinking rationale behind the dashboard, how it was built with an AI coding tool, and the complete prompt needed to reproduce it |

## 7. Licences and attribution

- AQHI data: Environmental Protection Department, HKSAR Government, via [DATA.GOV.HK](https://data.gov.hk).
- Pollutant concentration layer: [Air quality data by Open-Meteo.com](https://open-meteo.com/),
  licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); underlying data from the
  Copernicus Atmosphere Monitoring Service (CAMS), ECMWF.
- Basemap: Tiles © [Esri](https://www.esri.com/) — *Canvas World Light Gray Base*; source Esri, HERE,
  Garmin, © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright).
  Map library: [Leaflet](https://leafletjs.com).

  Esri's ArcGIS Online basemaps are used instead of OSM's official tiles because they impose no
  `Referer` requirement, which is what allows the page to work when opened directly as a local file.
  OSM is still credited as a data source of the basemap. See the [basemap notes](docs/basemap.md).
