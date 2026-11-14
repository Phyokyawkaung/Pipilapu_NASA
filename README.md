# Earth Analogs Explorer: team guide

NASA Space Apps 2026, "Earth analogs for Moon/Mars". An interactive 3D solar system that compares Moon and Mars sites with Earth, finds the most similar places on Earth, and plans a base on real terrain.

## Status (read this first)
**Real data (every value has a named source):**
- Earth grid for **Find on Earth** and the **Match / Profile / Data** tabs: terrain slope (NOAA ETOPO 2022) and air temperature (WorldClim 2.1).
- Moon/Mars target values in `docs/data/targets.json`: terrain slope, measured by `pipeline/build_targets.py` from LOLA (Moon) and MOLA-HRSC (Mars).
- Base planner terrain: NASA LOLA and USGS elevation patches, plus NASA PGDA sunlight and Earth-visibility maps at Shackleton.

**Not real yet:**
- **Named sites** now take their numbers from real data (`docs/data/sites.json`, built by `build_named_sites.py`). Still open: site coordinates are carried over from the old dataset and not yet checked against a gazetteer, and the geology tags are team-curated with no citation (shown, not scored).
- **Surface temperature** (`ts_mean`, `ts_range`): Earth from NASA MODIS MOD11C3 (2025, `prep_lst.py`), Moon from LRO Diviner (`prep_diviner.py`). Mars has no sourced value yet (no gridded NASA day/night surface temperature map found; MGS TES only has thermal inertia and albedo maps), so Mars sites are compared on terrain and landform only. Air temperature (WorldClim) is shown for Earth sites as context and never scored, because the Moon has no air.
- Base-planner **design parameters** (cell efficiency, losses and so on) are the user's own choices. They start empty. Agreed team values go into `docs/data/siting.json` with a source.

**Rule for everyone: no made-up numbers.** A value either has a named source, is derived from one (say how), or is typed by the user. If there is no source, the app says "No sourced value".

## Folder layout
```
Earth Analogus/
  docs/                    the website (named "docs" so GitHub Pages can publish it)
    index.html             the app: 3D view, panels, Match/Profile/Data tabs, Find on Earth, all styling
    planner.js             the Base plan tab (map, placement rules, roads, sizing)
    textures.js            fallback planet images
    assets/textures/       high-res planetary textures
    data/
      grid.json, *.bin     Earth grid (made by build_grid.py). Never edit by hand
      targets.json         Moon/Mars target values (made by build_targets.py). Never edit by hand
      sites.json           named sites for Match/Profile/Data (made by build_named_sites.py). Never edit by hand
      siting.json          Base planner: site facts, NASA constants, design parameters. Hand-edited, every value needs a source
      siting/<site>.json   real terrain patches (made by build_site_terrain.py). Never edit by hand
  pipeline/                Python scripts that make the files in docs/data
    build_grid.py          Earth grid from raw global rasters (edit its FEATURES list)
    build_targets.py       Moon/Mars target values, measured the same way as the Earth grid
    build_site_terrain.py  terrain patches for the base planner
    named_sites.json       hand-edited list of named sites: names, coordinates + source, tags + citation, body facts + source
    build_named_sites.py   named sites -> docs/data/sites.json, values read from the Earth grid and targets.json
    finder_pipeline.py     core maths: grid cells, slope, similarity, export
    raster_io.py           reads GeoTIFF files (fill values, scale factors)
    inspect_raster.py      prints what is inside a raster file
    make_demo_grid.py      writes FAKE grid files, for testing the setup only
    download_textures.py   downloads/refreshes planetary textures
    raw/                   downloaded NASA/Earth files (never committed to git)
    FIRST_RUN.md           notes from the first real-data run
  README.md
  .gitignore
```

## 1. Run the website (2 minutes)
You need Python 3.9 or newer (`py --version` on Windows) and internet, because three.js loads from a CDN.

1. Open a terminal in the project folder.
2. Start a local server: `py -m http.server 8000` (Mac/Linux: `python3 -m http.server 8000`).
3. Open http://localhost:8000/docs/ in Chrome or Edge. (If you started the server inside `docs`, use http://localhost:8000.)
4. Stop it with Ctrl+C. After changing a file, reload with Ctrl+F5.

Do not double-click `index.html`: browsers block `fetch` on `file://` pages, so the data would not load.

## 2. What the site does
- **Overview, Earth, Moon, Mars:** fly around the 3D solar system. Click a site marker to open its panel.
- **Panel tabs for a Moon/Mars site:**
  - **Match:** "Most similar places on Earth" (the top 5 cells of the real Earth grid), then the named analog sites.
  - **Profile and Data:** compare the site with the best place on Earth (default), another top place, or a named analog site, using the dropdown at the top.
  - **Base plan:** the base planner (section 3).
- **Find on Earth:** scores every 1-degree Earth cell against the site and paints the globe. Turn dimensions on or off and click a place to see why it scored as it did.
- **GIS:** top banner tool that displays continuous 1-degree planetary environmental rasters across Earth (8 layers), Moon (6 layers), and Mars (4 layers) with scientific colormaps, opacity slider, smooth interpolation toggle, dynamic range legends, and click-to-probe cell coordinates and readings.
- **Load data:** drop a dataset JSON (schema `earth-analogs/v1`) to replace the embedded named-site data.

How the score works: per feature, `1 - min(1, |difference| / scale)`; each dimension (terrain, landform, thermal) counts equally and its features share that weight; a cell with a missing feature is scored on the rest. The `scale` of each feature is set in `build_grid.py` (`FEATURES`) and is a starting guess that needs a written justification.

## 3. Base plan tab
Enter crew, days, power and design parameters. The planner places the habitat, solar field, batteries, landing pad, antenna, ice prospect, fission unit and science zone on real terrain with the rules in `RULES` (`docs/planner.js`), draws roads along the gentlest slopes, and sizes power, batteries and habitat volume.

- **Inputs are yours, not data.** They are remembered in your browser. "Clear my inputs" resets them. Percentages are typed as 0 to 100.
- **Missing inputs:** the "To complete the plan" list shows what is needed; click an item to jump to its box.
- **Site facts** come from `docs/data/siting.json`. A fact with no source shows "No sourced value" and a box for your own number.
- **Layers:** Relief, Slope, Sunlight and Earth view (the last two only where NASA maps exist, so far Shackleton). Click the map to move the habitat. **Save image** exports the map with a summary strip for slides.
- It is a screening estimate, not a base design. Formulas and rules are listed under "Rules used" and "Formulas" in the tab.

## 4. Make the data files
Install once: `pip install -r pipeline/requirements.txt` (numpy, rasterio, requests, pyhdf). Run commands from the project folder. Never edit the generated files by hand.

**Get the raw files:** `python pipeline/fetch_data.py` downloads every raw input (ETOPO, WorldClim, MODIS, Diviner; about 2.3 GB) into `pipeline/raw/` from the original servers, resumes broken downloads, and checks each file against `pipeline/data_manifest.json`. `--only earth` or `--only moon` for part of it, `--list` to see the list. MODIS needs a free NASA Earthdata login. The lead runs `--record` once and commits the manifest.

### 4a. Earth grid (`build_grid.py`)
1. Download global Earth rasters (elevation, temperature) from the sources listed in `FEATURES`. They must be GeoTIFFs in plain latitude/longitude (EPSG:4326) covering the whole world, with a pixel size that divides 1 degree evenly (such as 0.01). Fix them first if not:
   `gdalwarp -t_srs EPSG:4326 -te -180 -90 180 90 -tr 0.01 0.01 -r average in.tif pipeline/raw/dem.tif`
2. Put the files in `pipeline/raw/` under the names in `FEATURES` (`dem.tif`, `t_mean.tif`, `t_range.tif`, and `ts_mean.tif`, `ts_range.tif` from `prep_lst.py`, see 4a2).
3. Run `python pipeline/build_grid.py`. Files missing from `raw/` are skipped.
4. Spot-check a place you know: `python pipeline/build_grid.py --check 27.99,86.93` prints every feature there.

### 4a2. Earth surface temperature (`prep_lst.py`)
Source: NASA MODIS Terra **MOD11C3 v6.1** (monthly, 0.05 degree), 12 monthly `.hdf` files of one year, downloaded directly from Earthdata Search or the LP DAAC data pool (free Earthdata login). Each file holds the day and the night layer. Put them in `pipeline/raw/lst/`, then:
`pip install pyhdf` (once), then `python pipeline/prep_lst.py --hdf "pipeline/raw/lst/MOD11C3*.hdf"`
It writes `pipeline/raw/ts_mean.tif` ((day + night) / 2) and `pipeline/raw/ts_range.tif` (day - night), averaged over the months. Then rerun `build_grid.py`. Terra passes at about 10:30 and 22:30 local solar time; use Moon/Mars maps at similar local hours and say which in `--ts-src`. GeoTIFF exports (AppEEARS: `--day/--night --scale 0.02 --nodata 0`; NEO floating point: `--day/--night`) also work.

### 4a3. Moon surface temperature (`prep_diviner.py`)
Source: LRO Diviner Level 4 **Global Cumulative Product** (bolometric temperature, 0.5 degree bins, 0.25 h of local time, 2009-2015), https://pds-geosciences.wustl.edu/lro/lro-l-dlre-4-rdr-v1/lrodlr_1001/data/gcp/ . Download the `.tab` and `.xml` of the bands the sites need into `pipeline/raw/diviner/`: `00n10n` and `10s00s` (Tranquility), `90s80s` (Shackleton), about 156 MB each. Then:
`python pipeline/prep_diviner.py --tab "pipeline/raw/diviner/global_cumul_avg_cyl_*_002.tab"`
It writes `pipeline/raw/moon_ts_day.tif` (10:15-10:45 local time) and `moon_ts_night.tif` (22:15-22:45), in kelvin, the same hours as the MODIS overpasses. Then run `build_targets.py --site tranquility` and `--site shackleton` with `--ts-day`, `--ts-night`, `--ts-unit K` and `--ts-src`. Near the pole local time only says where the sun is in the sky, not whether it is up, so Shackleton's swing is small.

### 4b. Moon/Mars target values (`build_targets.py`)
`python pipeline/build_targets.py` reads a 111 km patch around each site from the real elevation models, averages it to 1853 m pixels (the pixel size of the Earth elevation model), measures slope, local relief and the hypsometric integral the same way as the Earth grid (shared functions in `finder_pipeline.py`), and writes `docs/data/targets.json`. Matching the method matters: slope depends strongly on pixel size.
- One site: `python pipeline/build_targets.py --site jezero`. Sites measured earlier are kept.
- The Shackleton read is about 125 MB and takes a few minutes.
- Terrain and landform only. Earth cells are 1° boxes, so they are narrower east-west than the square 111 km Moon/Mars patch away from the equator (about 29 km wide at 75° N); relief there is measured over a smaller area. Surface temperature for one site: `python pipeline/build_targets.py --site tranquility --ts-day DAY.tif --ts-night NIGHT.tif --ts-unit K --ts-src "DATASET NAME, local hours (URL)"`. It takes the mean of each map over the same 111 km patch and writes `ts_mean` and `ts_range`; other values for the site are kept. `--ts-unit` is required because lunar nights (about 100 K) cannot be told apart from degC automatically.

### 4c. Base planner terrain (`build_site_terrain.py`)
`python pipeline/build_site_terrain.py` writes `docs/data/siting/<site>.json` (elevation, plus NASA sunlight and Earth-visibility maps at Shackleton). Sites: shackleton, tranquility, jezero, olympus, valles. Options: `--site NAME`, `--km 8` (closer view), `--cell 20` (cell size in metres), `--dem FILE` (use a file you downloaded), `--debug` (show network messages).

### 4c2. Named sites (`build_named_sites.py`)
`python pipeline/build_named_sites.py` reads `pipeline/named_sites.json` and writes `docs/data/sites.json`. No numbers are typed in: each Earth site takes the values of the 1-degree grid cell that contains it, and each Moon/Mars site takes its values from `targets.json`. So a named site scores exactly like that cell in Find on Earth. Rerun it after `build_grid.py` or `build_targets.py`.
- To add a site: copy an entry in `named_sites.json`, give it an id, name, `lat`/`lon` with `coord_src` (where the coordinates come from), a summary, tags with `tags_src` (citation), `analog_for` (`["moon"]`, `["mars"]` or both), then rerun.
- Two sites in the same 1-degree cell get identical values.

### 4d. If NASA/USGS reads fail ("Read failed")
The scripts read only the part of a big file they need, over the internet, and retry on fresh connections. If a server keeps refusing:
1. Wait 15 to 30 minutes and run again with `--site NAME`.
2. Or download the file once in a browser and run with `--dem C:\path\file.tif`.
3. Run with `--debug` and send the output to whoever maintains the pipeline.

### 4e. Test the setup with fake data
`python pipeline/make_demo_grid.py` writes grid files marked `"demo": true`, and the site shows a DEMO banner. Never commit demo files over real ones.

## 5. Data files the website reads (docs/data)
- `grid.json`: manifest from the pipeline (cell size, shape, and each feature's label, unit, dimension, scale, source, resolution).
- `<feature>.bin`: one array per feature (float32, row-major, row 0 = 90N, column 0 = 180W, NaN = no data).
- `targets.json`: Moon/Mars target value per site and feature, each with its source text. Site ids: `shackleton`, `tranquility`, `jezero`, `olympus`, `valles`. `"demo": false` means real.
- `sites.json`: named sites in schema `earth-analogs/v1` (sources, metrics, bodies, sites, needs, scenarios). Generated.
- `siting.json`: constants with source and URL, design parameters (`assumptions`, empty until agreed), per-site facts (`null` = no sourced value).
- `siting/<site>.json`: terrain patch (`earth-analogs-siteterrain/v0`): `h` is base64 int16 metres; optional `sun` and `earth` are base64 uint8 percent (255 = no data).

## Common problems
| Symptom | Fix |
|---|---|
| Planets are flat colours, toast says textures not found | `textures.js` must sit next to `index.html` in `docs/`. |
| Stuck on the loading screen with "could not load the 3D library" | No internet, or the CDN is blocked. Reconnect and reload. |
| Data does not load, console shows a CORS or `file://` error | Use the local server (section 1). |
| `python` opens the Microsoft Store (Windows) | Use `py` instead of `python`. |
| `No module named rasterio` | `pip install rasterio`. If it fails on Windows, use Colab. |
| `not a global lat/lon raster` | Run the `gdalwarp` command in section 4a. |
| `pixels do not divide into 1-degree cells` | Resample to a pixel size like 0.01 degree. |
| Values look wrong or all the same | Check fill value (`nodata`), scale factor and `valid` range in `FEATURES`. |
| `Read failed` from a NASA/USGS file | Section 4d. |
| Old data or old look still showing | Hard reload (Ctrl+F5). |
| Match tab says "Not available yet" | Run `build_grid.py` (grid files) or `build_targets.py` (targets), and use the local server. |
| Base plan says "No sourced value" | Expected. Type your own number in the box, or add a sourced value to `siting.json`. |
| Base plan map missing, only a schematic | Run `build_site_terrain.py --site NAME`. |

## Teamwork rules
- **Git:** `git pull` before you start; then `git add` the files you changed (use `git status` to check nothing big is staged), `git commit -m "what you did"`, `git push`.
- **Commit:** code, `docs/`, and the small generated files in `docs/data/` (the largest, `siting/shackleton.json`, is under 1 MB). **Never commit** raw downloads (`pipeline/raw/`, any `.tif`, `.zip`). Keep large raw files on Drive.
- **One owner for `docs/index.html`** at a time. It is one big file and merges badly. Tell the team before you edit it.
- Do not hand-edit generated files in `docs/data/`; rerun the pipeline script. `siting.json` is the one hand-edited data file, and every value needs a source.
- Record the source name, URL and access date for every real dataset.
- Check numbers from free AI tools against the source page before using them. They can invent plausible values.
- **Style:** flat and neutral. Amber means a Moon/Mars target, blue means an Earth analog. No emojis. The styling is the "UI overhaul" block at the end of the `<style>` in `index.html`.

## Free compute (Google Colab)
Big rasters can be heavy for a laptop. In a Colab notebook: upload the raw files to Google Drive, then
```
from google.colab import drive; drive.mount('/content/drive')
!pip install rasterio
!git clone <your repo url>
%cd <repo folder>
!python pipeline/build_grid.py --raw /content/drive/MyDrive/raw --out docs/data
```
Download `docs/data` afterwards and commit it.

## Publish for free (GitHub Pages)
Push the repo to GitHub, then Settings, Pages, deploy from branch `main`, folder `/docs`. The site appears at a github.io address after a minute or two. (Check GitHub's current instructions if the menus differ.) The site loads three.js from the CDN, so viewers need internet.
