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
| Interactive map of all 18 stations | Leaflet.js (CDN) + Esri Canvas **Light** Gray Base tiles — works over both `file://` and `http(s)`, see the [basemap notes](docs/basemap.md) |
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
  Both routes work identically, including the map — which is why the basemap is Esri rather than
  OpenStreetMap (see the [basemap notes](docs/basemap.md)). The two APIs are `CORS *` and keyless, so
  nothing needs a local server; the only `file://` casualty is the OSM tile `Referer` requirement.
- **GitHub Pages (your own deployment)**: push `index.html` to *your* GitHub repository, then enable
  Pages via **Settings → Pages → Deploy from a branch** (select the branch and `/ (root)`). Your
  dashboard will go live at `https://<your-username>.github.io/<repo-name>/` — replace the
  placeholders with your own GitHub username and repository name.
  (The live-demo link at the top of this README is the author's own deployment.)

Desktop browsers assumed (no mobile optimisation, by design).

## 4. Known limitations

Worth stating plainly, because several of these are consequences of the constraints in §2 rather than
oversights:

| Limitation | Consequence |
|---|---|
| Pollutant values are **modelled** (CAMS via Open-Meteo), not measured | The dashboard is a visualisation aid, not an authority. [EPD's own site](https://www.aqhi.gov.hk) publishes the official analyser readings |
| The station join key is the **exact English station name** | If EPD renames a station, that station silently disappears from the map and table rather than raising an error |
| Coordinates are **hardcoded**, not fetched | New or relocated stations will not appear until the `STATIONS` array is updated by hand |
| Refresh interval is 300 s, but EPD publishes AQHI **hourly** | The map can lag the official figure by up to one publish cycle; the publish time is shown in each popup so the lag is visible |
| Basemap stops at **zoom 16** | `Canvas/World_Light_Gray_Base` has no data beyond it; street-level zoom on Esri's other services would be needed |
| Desktop only | No mobile or tablet layout; the sidebar is a fixed-width scrolling column |
| Four external dependencies | Leaflet (unpkg CDN), two live APIs and the Esri tile service must all be reachable; offline use shows tiles and data failing together |
| Esri attribution must stay visible | Removing or shortening the credit breaches Esri's terms — see the [basemap notes](docs/basemap.md) |

## 5. Documentation

| Document | What it covers |
|---|---|
| **[Basemap notes](docs/basemap.md)** | Why the basemap is Esri rather than OpenStreetMap — the `file://` `Referer` trap that silently returns `HTTP 200` with a refusal image, the three fixes that do *not* work, the `{z}/{y}/{x}` and `maxZoom: 16` traps, and how to verify a tile layer by byte size instead of by eye |
| **[Vibe-coding guide](docs/vibe-coding.md)** | The teaching pack for reproducing this project with an AI coding tool: the design-thinking rationale, how the dashboard was actually built, the complete copy-paste prompt, the learning outcomes, and the assessment rubric |

## 6. Licences & attribution

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
