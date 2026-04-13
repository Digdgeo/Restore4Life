# Restore4Life — Danube Wetlands Hydroperiod App

Interactive widget for hydroperiod analysis of Danube basin wetlands, powered by [ndvi2gif](https://github.com/Digdgeo/Ndvi2Gif).

## Features

- Leafmap/geemap interactive map centred on the Danube basin
- Wetland selector from a user-supplied shapefile
- Dataset selection: Sentinel-2, Landsat, MODIS
- Water index selection: MNDWI, NDWI, AWEI, AWEInsh, WI2015
- Adjustable threshold and cloud filter
- Full hydroperiod computation via `HydroperiodAnalyzer`:
  - Bands: `hydroperiod`, `normalized`, `valid_days`, `first_flood_doy`, `last_flood_doy`, `irt`
  - Multi-cycle analysis (one image per hydrological year)
- Export results to Google Drive

## Setup

```bash
pip install ndvi2gif geemap ipyevents geopandas
```

## Required data

Place the following shapefiles in the `data/` folder:

| File | Description |
|------|-------------|
| `danube_basin.shp` | Danube basin boundary (map centering) |
| `wetlands.shp` | Wetland polygons to analyse |

## Usage

```python
import ee
import geemap
from restore4life import HydroperiodApp

ee.Initialize()

Map = geemap.Map()
HydroperiodApp(Map)
Map
```

See `notebooks/demo.ipynb` for a full walkthrough.
