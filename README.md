# Restore4Life — Danube Wetlands HydroApp

Widget interactivo para el análisis del hidroperiodo en humedales de la cuenca del Danubio, apoyado en [ndvi2gif](https://github.com/Digdgeo/Ndvi2Gif) y ejecutado sobre Google Earth Engine.

## Qué hace

Panel plegable en el mapa con 5 pestañas:

- **Hydroperiod** — cálculo de ciclos hidrológicos (S2, Landsat, MODIS) con índices de agua configurables y umbral ajustable.
- **Anomalies** — anomalías respecto a la media del periodo o al histórico completo.
- **TWI** — Topographic Wetness Index (MERIT Hydro o híbrido NASADEM + MERIT).
- **Stats** — estadísticas zonales por punto o polígono sobre cualquier producto computado.
- **Export** — exportación de resultados a Google Drive.

Selección de ROI por dropdown (humedales del Danubio o sitios eLTER), subida de shapefile/GeoJSON/ZIP, o dibujo directo sobre el mapa.

Ver [`FEATURES.md`](FEATURES.md) para el desglose completo.

## Instalación

```bash
pip install ndvi2gif geemap ipyevents geopandas
```

## Datos

Los siguientes archivos deben estar en la raíz del repo (ya incluidos):

- `DRBD_2021.shp` (+ auxiliares) — cuenca del Danubio
- `humedales_danubio.shp` (+ auxiliares) — humedales a analizar
- `elter_danube.geojson` — sitios eLTER en la cuenca (opcional; regenerar con `scripts/build_elter_danube.py`)
- `logo.png` — logo del widget

## Uso

```python
import ee
import geemap
from restore4life import HydroperiodApp

ee.Initialize()

Map = geemap.Map()
HydroperiodApp(Map)
Map
```

Ver `notebooks/demo.ipynb` para un walkthrough completo.
