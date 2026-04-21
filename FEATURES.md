# Restore4Life — HydroApp: Guía de funcionalidades

Widget interactivo basado en `leafmap`/`geemap` para el análisis del hidroperiodo en humedales de la cuenca del Danubio. Utiliza [`ndvi2gif`](https://github.com/Digdgeo/Ndvi2Gif) (`NdviSeasonality` + `HydroperiodAnalyzer`) como motor de cálculo sobre Google Earth Engine.

Punto de entrada: `restore4life.HydroperiodApp(Map)` — inyecta un panel plegable (esquina superior derecha del mapa) con 5 pestañas.

---

## 1. Mapa base y capas de contexto

Al inicializar el widget sobre un mapa de `geemap`:

- **Basemap**: `Esri.WorldImagery` (imagen satelital).
- **Danube Basin** (`DRBD_2021.shp`): contorno de la cuenca en dorado, sin relleno. Se usa también para:
  - Centrar el mapa (`fit_bounds` sobre los límites de la cuenca).
  - Limitar paneo/zoom (`max_bounds` + `min_zoom = 5`) — no se puede salir de la zona.
- **Wetlands** (`humedales_danubio.shp`): humedales del Danubio en azul translúcido.
- **eLTER sites (Danube)** (`elter_danube.geojson`, opcional): sitios eLTER que intersectan la cuenca, en verde. Se genera/actualiza con `scripts/build_elter_danube.py` (consulta DEIMS-SDR).

Todo se carga localmente vía GeoJSON — cero llamadas a EE en el arranque.

---

## 2. Selección de ROI (región de interés)

Cuatro formas de definir la ROI, mutuamente excluyentes:

| Método | Cómo |
|---|---|
| **Dropdown de humedales** | Lista alfabética desde `humedales_danubio.shp` (columna de nombre autodetectada). |
| **Dropdown eLTER** | Sitios eLTER filtrados por intersección con la cuenca. |
| **Upload** | Sube `.geojson`, `.json` o `.zip` (shapefile). Se reproyecta a EPSG:4326. |
| **Dibujar en el mapa** | Botón *Draw on map* activa `DrawControl` para polígono o rectángulo. |

Botón **Clear ROI** resetea selección, dibujo y dropdowns. Seleccionar una fuente desactiva las otras automáticamente.

---

## 3. Pestaña **Hydroperiod** — cálculo principal

Configuración del cómputo:

- **Dataset**: Sentinel-2 (desde 2017), Landsat (desde 1984), MODIS (desde 2000). Cambiar dataset ajusta años mínimos, índices de agua disponibles y escala por defecto de export.
- **Start/End year**: rango de años hidrológicos.
- **Max clouds %**: filtro de nubes (slider 0–100).
- **Water index** (según sensor):
  - S2 / Landsat: `mndwi`, `ndwi`, `awei`, `aweinsh`, `wi2015`
  - MODIS: `mndwi`, `ndwi`
- **Threshold**: umbral de binarización agua/no-agua (-1.0 a 1.0, paso 0.01).
- **Band**: banda a visualizar tras el cómputo — `normalized`, `hydroperiod`, `valid_days`, `first_flood_doy`, `last_flood_doy`, `irt`.
- **Hyd. year**: se rellena automáticamente tras *Compute* con los ciclos disponibles (formato `YYYY/YYYY+1`).

Botones:

- **Compute** — 3 fases: construye `NdviSeasonality` → `HydroperiodAnalyzer` → `compute_all_cycles(index, threshold)`. Deja un ciclo por año hidrológico.
- **Show** — añade al mapa la banda seleccionada del año elegido, con paleta específica por banda. Para `irt` computa `compute_irt_image()` bajo demanda y lo cachea.
- **Reset** — limpia estado, capas computadas, ROI, dibujos y selects.

---

## 4. Pestaña **Anomalies**

Requiere *Compute* previo en la pestaña Hydroperiod.

- **Reference**:
  - `Period mean` — media de los años seleccionados en el rango actual.
  - `Historical` — media del archivo histórico completo del sensor.
- **Compute anomalies** — llama a `HydroperiodAnalyzer.compute_anomalies(cycles, reference)`, devuelve `{'mean': ee.Image, 'anomalies': {year: ee.Image}}`.
- **Show mean** — añade al mapa la media de referencia (paleta `mean_hydroperiod`).
- **Show anomaly** — añade la anomalía del año elegido (paleta divergente rojo→verde, rango ±180 días).

---

## 5. Pestaña **TWI** (Topographic Wetness Index)

TWI = ln(a / tan β), donde *a* = área acumulada aguas arriba por unidad de contorno y *β* = pendiente local.

- **DEM source**:
  - `MERIT Hydro (~90 m)` — pendiente y acumulación de MERIT, internamente consistente.
  - `Hybrid 30 m` — NASADEM (30 m) para pendiente + MERIT `upa` reproyectado para acumulación. La acumulación sigue siendo ~90 m (sink-filling + D8 real a 30 m no es práctico en GEE).
- **Compute TWI** — calcula sobre la ROI actual.
- **Show** — añade al mapa con paleta azul (rango 2–20).
- **Export to Drive** — exporta el TWI actual como GeoTIFF. Escala configurable (30, 60, 90, 250, 500 m).

---

## 6. Pestaña **Stats** — estadísticas zonales

Calcula estadísticos por feature sobre cualquier producto ya computado en las otras pestañas.

- **Upload**: `.geojson` / `.json` / `.zip` con **puntos** o **polígonos** (autodetecta tipo de geometría y columna de nombre).
- **Product**:
  - `Hydroperiod (año + banda actuales)`
  - `IRT`
  - `Mean hydroperiod` (de Anomalies)
  - `Anomaly` (año actual en Anomalies)
  - `TWI`
- **Scale (m)**: resolución de muestreo. Se autoajusta según producto/sensor (TWI → 90 m, S2 → 10 m, Landsat → 30 m, MODIS → 500 m).
- **Reducer**:
  - Puntos → `first()` (valor del píxel).
  - Polígonos → combinación de `mean`, `median`, `min`, `max`, `stdDev`, `count`.

Botones:

- **Compute stats** — lanza `reduceRegions` y renderiza la tabla resultante en el panel (pandas DataFrame). Avisa si hay >1000 features.
- **Save CSV** — guarda `stats_YYYYMMDD_HHMMSS.csv` en el cwd del notebook.
- **Export to Drive** — export server-side vía `ee.batch.Export.table.toDrive` (recomendado para datasets grandes).

---

## 7. Pestaña **Export** — exportar ciclos completos

- **Drive folder**: nombre de la carpeta destino en Google Drive (por defecto `restore4life_hydroperiod`).
- **Scale (m)**: 10 / 20 / 30 / 100 / 250 / 500.
- **Export** — lanza una task `ee.batch.Export.image.toDrive` por cada ciclo computado (nombradas `hydroperiod_<site>_<yyyy>_<yyyy+1>`).

---

## 8. UI — comportamiento del panel

- Botón flotante con icono 💧 (`tint`) — esquina superior derecha del mapa.
- Se expande al hacer click o al pasar el ratón por encima.
- Botón ✕ (`close`) cierra el panel y elimina el control del mapa.
- Paletas de colores definidas por banda en el diccionario `_VIS` del módulo.

---

## 9. Datos requeridos en la raíz del repo

| Archivo | Descripción | Fuente |
|---|---|---|
| `DRBD_2021.shp` (+ .dbf/.prj/.shx/.cst) | Cuenca del Danubio | provisto |
| `humedales_danubio.shp` (+ auxiliares) | Humedales a analizar | provisto |
| `elter_danube.geojson` | Sitios eLTER (opcional) | generar con `scripts/build_elter_danube.py` |
| `logo.png` | Logo Restore4Life (cabecera del widget) | provisto |

---

## 10. Uso mínimo

```python
import ee
import geemap
from restore4life import HydroperiodApp

ee.Initialize()

Map = geemap.Map()
HydroperiodApp(Map)
Map
```

Ver `notebooks/demo.ipynb` para el walkthrough completo.
