---
title: "L6 – Urban Data Sources: Cadastre, OSM, EO & IoT"
layout: default
---

# Lecture 6 — Urban Data Sources: Cadastre, OSM, Earth Observation & IoT

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 2 · Lecture 6 of 18**  
> 🎯 **Objective:** Identify, evaluate, and integrate the main urban data source categories for a UDT

---

## 🎬 Lecture Overview

In this lecture you will learn:
- The four main urban data source families: Cadastre, OSM, Earth Observation, IoT
- Strengths, limitations, and formats of each
- How to integrate heterogeneous sources into a unified PostGIS database
- Data quality and update-cycle considerations

---

## 📖 Part 1 — Official and Crowdsourced Data (~4 min)

### 6.1 Why Multiple Sources?

No single data source covers all UDT needs. A city digital twin typically requires:

| Layer | Best Source |
|-------|------------|
| Building footprints & attributes | Cadastre |
| Road network, POIs, land use | OSM |
| Land surface temperature, vegetation | Earth Observation (Sentinel) |
| Air quality, noise, occupancy | IoT sensors |
| Population, demographics | Census / official statistics |

### 6.2 Cadastral Data

**Cadastres** are official government registries of land parcels, ownership, and property attributes. They are typically the most **authoritative and geometrically precise** urban data source.

**Typical attributes:**
- Parcel ID, area, owner (often anonymized)
- Building footprint, number of floors, construction year
- Land use classification, zoning
- Assessed value

**Formats and access:**
- GML, SHP, GeoPackage, or national formats (e.g., Topografische Kaart in the Netherlands)
- Open in many countries (Netherlands PDOK, UK OS OpenData, France data.gouv.fr)
- Accessed via WFS endpoints or bulk download

**Limitations:**
- Update lag (months to years behind reality)
- Varying attribute completeness across municipalities
- Rarely includes 3D geometry (LoD2+ requires separate surveys)

### 6.3 OpenStreetMap (OSM)

**OpenStreetMap** is the world's largest collaborative geospatial database. For UDTs it provides:

- **Road network** (highly accurate in urban areas)
- **Points of Interest** (shops, schools, hospitals, transit)
- **Land use polygons** (parks, industrial, residential)
- **Building outlines** (in well-mapped cities)

**Access methods:**

```bash
# Download city extract via osmium
osmium extract --bbox 6.85,52.20,6.95,52.25 netherlands-latest.osm.pbf \
  -o enschede.osm.pbf

# Convert to GeoPackage for PostGIS import
ogr2ogr -f GPKG enschede.gpkg enschede.osm.pbf
```

**Limitations:**
- Quality is crowd-dependent (excellent in major cities, sparse in rural areas)
- No official land ownership data
- Buildings often lack height/floor attributes

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Best match**

You need precise building footprints with legal parcel boundaries for a housing affordability analysis. Which source is most appropriate?

- A) OpenStreetMap
- B) **Cadastral data** ✅
- C) Sentinel-2 satellite imagery
- D) IoT sensor network

**Question 2 — Multiple select**

Which of the following data types can be derived from **Sentinel-2** satellite imagery? (Select all that apply)

- A) **NDVI (vegetation index)** ✅
- B) Road centerlines
- C) **Land use / land cover classification** ✅
- D) Air temperature at street level
- E) **Impervious surface detection** ✅

**Question 3 — True or False**

IoT sensor data is typically the most geometrically precise source for building footprints.

- A) True
- B) **False** ✅ *(IoT provides temporal/environmental data, not geometric precision)*

---

## 📖 Part 2 — Earth Observation & IoT Integration (~4 min)

### 6.4 Earth Observation: Sentinel Program

The **Copernicus Sentinel** satellites provide free, systematic, multi-temporal imagery for UDT applications:

| Satellite | Resolution | Key UDT Applications |
|-----------|-----------|----------------------|
| **Sentinel-1** | 10–20m (SAR) | Building damage, flood mapping, subsidence |
| **Sentinel-2** | 10–60m (multispectral) | NDVI, land use, impervious surface, urban growth |
| **Sentinel-3** | 300–1000m | Land Surface Temperature (LST), sea level |
| **Sentinel-5P** | ~3.5km | Air quality (NO₂, PM2.5, CO) |

**Accessing Sentinel data:**

```python
# Using sentinelsat Python library
from sentinelsat import SentinelAPI

api = SentinelAPI('user', 'password', 'https://scihub.copernicus.eu/dhus')
products = api.query(
    area='POLYGON((6.85 52.20, 6.95 52.20, 6.95 52.25, 6.85 52.25, 6.85 52.20))',
    date=('20240601', '20240630'),
    platformname='Sentinel-2',
    cloudcoverpercentage=(0, 20)
)
api.download_all(products)
```

**Common EO-derived indicators for UDTs:**

```
NDVI = (NIR - Red) / (NIR + Red)   → Vegetation density
NDBI = (SWIR - NIR) / (SWIR + NIR) → Built-up intensity
LST  = Thermal band + emissivity correction → Surface temperature
```

### 6.5 IoT Sensor Networks

IoT sensors provide what satellites cannot: **high temporal resolution, real-time, street-level data**.

**Common sensor types in urban UDTs:**

| Sensor | Measures | Example Use |
|--------|---------|-------------|
| Weather station | Temperature, humidity, wind | Microclimate mapping |
| Air quality (low-cost) | PM2.5, NO₂, CO | Pollution exposure |
| Traffic counter | Vehicle count, speed | Mobility modeling |
| Smart meter | Energy consumption | Building energy efficiency |
| Water pressure sensor | Pipe pressure | Leak detection in water networks |
| CCTV / depth camera | People flow, occupancy | Public space usage |

**Integration challenges:**
- **Data format heterogeneity** (JSON, CSV, binary, proprietary protocols)
- **Temporal resolution mismatch** (1-second sensor vs. yearly cadastre)
- **Spatial alignment** — sensors have coordinates but variable GPS precision
- **Data gaps** — sensors fail, go offline, get moved

### 6.6 Building a Unified ETL Pipeline

The key to integrating all sources is a robust **Extract-Transform-Load (ETL)** pipeline:

```
[Cadastre WFS] ─────────────────────┐
[OSM .pbf] ────── ogr2ogr/osmium ───┤
[Sentinel .SAFE] ─── GDAL/rasterio ─┼──→ PostGIS (unified schema)
[IoT MQTT feed] ─── Python/FIWARE ──┤
[Census CSV] ─────── GeoPandas ─────┘
```

**Key tools:**
- `GDAL/OGR` — universal format converter (200+ formats)
- `GeoPandas` — Python spatial dataframes for data wrangling
- `osmium-tool` — OSM processing
- `rasterio / rio-cogeo` — raster ingestion and COG creation
- `Apache Airflow / Prefect` — workflow orchestration for scheduled updates

**Unified PostGIS schema principle:**

```sql
-- Every imported layer gets a source_metadata table
CREATE TABLE data_sources (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    source_type VARCHAR(50),  -- 'cadastre', 'osm', 'sentinel', 'iot'
    update_freq VARCHAR(50),  -- 'static', 'daily', 'real-time'
    crs         INTEGER,      -- EPSG code
    last_updated TIMESTAMPTZ
);
```

---

## 🔑 Key Takeaways

1. Urban Digital Twins require **four complementary data source families**: cadastre, OSM, Earth Observation, and IoT.
2. **Cadastral data** is authoritative for legal boundaries; **OSM** provides network and POI richness; **Sentinel** enables time-series spatial analysis; **IoT** provides real-time, street-level readings.
3. A **unified ETL pipeline** (GDAL + GeoPandas + PostGIS) is essential to harmonize heterogeneous sources into a single queryable database.
4. The most common integration challenges are **format heterogeneity**, **temporal resolution mismatch**, and **CRS inconsistency**.

---

## 📎 References & Further Reading

- Copernicus Open Access Hub: https://scihub.copernicus.eu/
- OSM Wiki: https://wiki.openstreetmap.org/
- GDAL Documentation: https://gdal.org/
- FIWARE Smart Data Models: https://smartdatamodels.org/
- Netherlands PDOK (open cadastral data): https://www.pdok.nl/

---

[← L5](lecture-05.md) · [Module 2 Index](index.md) · [Next: Module 3 →](../module-3/index.md)
