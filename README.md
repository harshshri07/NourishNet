# NourishNet

A one-stop food assistance tool for Maryland and the DC metro area. Built with React, TypeScript, and Leaflet. Connects families to food, donors to organizations, and volunteers to opportunities.

Live site: [https://protocorn.github.io/NAFSI_Track2/](https://protocorn.github.io/NAFSI_Track2/)


## Prerequisites

- Node.js 18 or later
- npm 9 or later
- Python 3.12 (only needed for the data pipeline and chatbot)


## Quick Start (Frontend Only)

The React app runs as a static site with no backend required. All data is pre-built into JSON files under `nourishnet/public/data/`.

```bash
cd nourishnet
npm install
npm run dev
```

Open http://localhost:5173 in your browser.


## Build for Production

```bash
cd nourishnet
npm run build
npm run preview
```

The built files go to `nourishnet/dist/`. The GitHub Actions workflow in `.github/workflows/deploy-pages.yml` handles automatic deployment to GitHub Pages on push to main.


## Project Structure

```
nourishnet/              React app (Vite + TypeScript + Tailwind)
  src/
    pages/               Consumer, Donor, Volunteer, Home, About
    components/          NourishMap, DonorChatbot, FoodInsecurityOverlay, etc.
    hooks/               useCatalog, useDonorCatalog, useGeocode, useGeolocation
    utils/               geo, hours, leafletIcons, pointInPolygon
    contexts/            LanguageContext (EN/ES/FR/AM)
    i18n/                translations.ts
    types.ts             Place, DonorPlace, Catalog, DonorCatalog, CountyStat
  public/data/           catalog.json, donor_catalog.json, md_counties.geojson

Chatbot/                 FastAPI chatbot backend (optional)
  chatbot.py             RAG chatbot using Google Gemini
  data/                  Unified CSV files for chatbot context
  requirements.txt       Python dependencies
  Procfile               Heroku deployment config

data/
  raw_data/              Original source files
  cleaned/               After clean_data.py and clean_data_pass2.py
  unified/               Final consumer/, donor/, volunteer/ catalogs

config/
  sources.yaml           Data source registry (URLs, types, query params)

helpers/
  County_List.py         Maryland/DC county name mappings

clean_data.py            Stage 1: filter to MD/DC, deduplicate, merge
clean_data_pass2.py      Stage 2: normalize phones, strip whitespace, fix types
finalize_unified.py      Stage 3: build unified directory, standardize columns
geocode_catalog.js       Stage 4: batch geocode missing lat/lng coordinates
```


## Data Pipeline

The data pipeline transforms raw CSV/JSON files from 25+ sources into two JSON catalogs that the React app consumes. You do not need to run the pipeline to use the app (the catalogs are already built), but here is how it works if you want to rebuild from source.

### Step 1: Clean raw data

```bash
python clean_data.py
```

Reads files from `data/raw_data/`, filters to Maryland and DC only, removes duplicates, and writes cleaned files to `data/cleaned/`.

### Step 2: Normalize

```bash
python clean_data_pass2.py
```

Strips whitespace, normalizes phone numbers to (XXX) XXX-XXXX format, converts currency/percentage strings to numbers, and standardizes county names.

### Step 3: Unify

```bash
python finalize_unified.py
```

Builds the `data/unified/` directory with separate folders for consumer, donor, and volunteer data. Standardizes column names to snake_case.

### Step 4: Geocode (optional)

```bash
node geocode_catalog.js
```

Batch geocodes locations missing latitude/longitude using the Google Maps API. Results are cached in `data/unified/donor/pantry_geocache.json` to avoid redundant API calls. Requires a Google Maps API key.

### Step 5: Build JSON catalogs

The unified CSV files are converted to `catalog.json` and `donor_catalog.json` and placed in `nourishnet/public/data/` for the React app to load at runtime.


## Scraping

Raw data is collected by two scraper families — **active** (automated, checkpointed, run on a schedule) and **passive** (one-off scripts for static or semi-static sources). All scrapers write to `data/sources/` and feed into the data pipeline described above.

### Active Scrapers — `scrappers/active_scrappers/`

Driven by `config/sources.yaml`. Each source entry declares its type, URL, and query parameters. The CLI reads the config, checks a checkpoint file to avoid redundant downloads, and only fetches when new data is available.

```bash
# Run from repo root
python -m scrappers.active_scrappers.cli --source snap_retailer_locator
python -m scrappers.active_scrappers.cli --source caroline_food_pantries_flyer
python -m scrappers.active_scrappers.cli --force-export   # ignore checkpoint
```

**Supported source types:**

| Type | How it works |
|---|---|
| `arcgis_feature_layer` | Paginates an ArcGIS REST Feature Layer via Playwright's `APIRequestContext`. Reads the SNAP Locator subtitle date to detect new snapshots before downloading. Writes a dated CSV + manifest JSON. |
| `flyer_pdf_page` | Loads a web page in headless Chromium, reads the "Last Updated" date, downloads the linked PDF, extracts text with `pypdf`, and applies heuristic phone-block parsing to produce a structured locations CSV. |

**Checkpoint system:** Each source directory gets a `.export_state.json` file recording the last successfully exported snapshot date. The CLI skips the download if the live site date matches or is older than the checkpoint, keeping bandwidth usage minimal.

### Passive Scrapers — `scrappers/passive_scrappers/`

One-off scripts for sources that don't change frequently or require manual steps:

| Script | Source | Output |
|---|---|---|
| `scraper_211md_v2.py` | 211 Maryland search portal | Food pantry records with address, phone, categories |
| `pg_healthy_foods_export.py` | PG County healthy food store dataset | Grocery/meat/seafood stores with address and category |
| `filter_food_only.py` | Post-processing filter | Strips non-food records from 211MD output |
| `open_feeding_america_county_pages.py` | Feeding America map | Opens county pages for manual data collection |

### Config — `config/sources.yaml`

All active scraper sources are registered here. Each entry includes:

```yaml
sources:
  - id: snap_retailer_locator
    name: SNAP Retailer Locator
    type: arcgis_feature_layer
    layer_url: https://...
    locator_page_url: https://...
    where: "State = 'MD'"
    out_fields: "*"
    page_size: 1000

  - id: caroline_food_pantries_flyer
    name: Caroline County Food Pantries Flyer
    type: flyer_pdf_page
    page_url: https://...
    pdf_text_pages: [0]          # page indices to extract (0 = English, skip Spanish)
    flyer_structured_rows: true  # run heuristic phone-block parser
```

### Scraper Dependencies

```bash
pip install playwright pypdf pyyaml
python -m playwright install chromium
```

Playwright is only needed for the active scrapers. The passive scrapers use `requests` and `beautifulsoup4`.

---

## Data Sources

| Source | Records | Type |
|--------|---------|------|
| SNAP Retailer Locator (USDA) | 4,196 | ArcGIS API scrape |
| Food Pantries (merged from 5 sources) | 1,265 | CSV download + PDF extraction |
| Capital Area Food Bank | 382 | JSON/CSV download |
| 211 Maryland | 378 | Web scrape |
| Census Tracts (PG County) | 283 | CSV download |
| Market Data (PG County) | 189 | CSV download |
| Feeding America Maryland | 120 | CSV download |
| Farmers Markets | 21 | CSV download |

Additional geospatial data: Maryland county boundaries GeoJSON for the food insecurity choropleth overlay.


## Chatbot Setup (Optional)

The donor chatbot is a separate FastAPI server that uses Google Gemini for RAG (retrieval-augmented generation). It is not required for the main app to work.

### 1. Install Python dependencies

```bash
cd Chatbot
pip install -r requirements.txt
```

### 2. Set up your API key

Create a `.env` file in the project root:

```
GEMINI_API_KEY=your-gemini-api-key-here
```

You can get a free API key from [Google AI Studio](https://aistudio.google.com/apikey).

### 3. Run the chatbot server

```bash
cd Chatbot
python chatbot.py
```

The server starts on http://localhost:8000. The Vite dev server automatically proxies `/api/chat` requests to this backend (configured in `nourishnet/vite.config.ts`).


## Features

### For Families (Find Food)
- Search by ZIP, city, county, or address
- Filter by place type, county, and day of week
- Interactive map with clustered markers
- "I Need Food Now" emergency button
- One-tap phone calls and directions
- Available in English, Spanish, French, and Amharic

### For Donors
- All donation-accepting locations on a searchable map
- Food insecurity choropleth overlay (county-level)
- County hover info showing food-insecure population
- AI chatbot for natural-language donation questions
- Impact panel with food insecurity statistics

### For Volunteers
- Volunteering opportunities linked to food banks and pantries
- Filter by county and weekday/weekend availability
- Interest form that opens a pre-filled email to the organization

### General
- Mobile-responsive layout
- Keyboard accessible
- No account required, no personal data collected
- Static site deployment (GitHub Pages)


## Tech Stack

- React 19, TypeScript, Vite 8
- Tailwind CSS 4
- Leaflet + react-leaflet (maps and marker clustering)
- MapLibre GL (choropleth overlay)
- React Router 7
- Nominatim / OpenStreetMap (free geocoding)
- FastAPI + Google Gemini 2.0 Flash (chatbot, optional)
- GitHub Actions (CI/CD to GitHub Pages)


## Team

Built by students at the University of Maryland for the NourishNet Data Challenge 2026.

- Rishabh Ranka
- Harsh Shrishrimal
- Sahil CHordia
- Rishika Thakre
- Jiten Bhalavat


## License

Open source. Built exclusively with open source tools and libraries.
