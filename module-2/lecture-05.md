---
title: "L5 – Serving Geo-Data: GeoServer, TiTiler & OGC APIs"
layout: default
---

# Lecture 5 — Serving Geo-Data: GeoServer, TiTiler & OGC APIs

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 2 · Lecture 5 of 18**  
> 🎯 **Objective:** Configure a spatial data serving layer using GeoServer and TiTiler to feed a UDT front-end

---

## 🎬 Lecture Overview

In this lecture you will learn:
- The role of a spatial data server in the UDT stack
- GeoServer: publishing vector and raster layers via OGC standards
- TiTiler: serving Cloud-Optimized GeoTIFFs as map tiles
- When to use each tool and how they complement each other

---

## 📖 Part 1 — The Data Serving Layer (~4 min)

### 5.1 Why Do We Need a Geo-Data Server?

A spatial database (PostGIS) stores and queries data efficiently — but it is not designed to serve thousands of concurrent HTTP requests for map tiles. The **serving layer** sits between the database and the web front-end:

```
PostGIS (store & query)
        ↓
GeoServer / TiTiler (serve via HTTP)
        ↓
Mapbox GL JS / Cesium / Leaflet (display)
```

The serving layer handles:
- Rendering or packaging data into map tiles
- Enforcing access controls
- Reprojecting on-the-fly
- Responding to OGC API requests

### 5.2 GeoServer: The Swiss Army Knife of Geo-Data Serving

**GeoServer** is an open-source Java server that publishes geospatial data from PostGIS, shapefiles, GeoTIFFs, and other sources via OGC standards.

**What GeoServer serves:**

| Service | Output | Use in UDT |
|---------|--------|-----------|
| WMS | Map images (PNG/JPEG) | Background layers, styled maps |
| WFS | GeoJSON / GML features | Interactive vector queries |
| WCS | Raw raster data | DEM, LST grids for analysis |
| WPS | Processing results | On-demand spatial analysis |
| OGC API Features | REST/JSON | Modern front-end consumption |

**Key GeoServer concepts:**
- **Workspace** — logical grouping (e.g., "water_dept", "planning_dept")
- **Store** — connection to a data source (PostGIS, file)
- **Layer** — a published dataset with a style (SLD)
- **Layer Group** — combine multiple layers (like a thematic map)

### 5.3 Styling with SLD

GeoServer uses **Styled Layer Descriptor (SLD)** — an OGC XML standard — to define how layers look. Example: style buildings by height with a graduated color ramp:

```xml
<Rule>
  <RasterSymbolizer>
    <ColorMap type="intervals">
      <ColorMapEntry color="#ffffcc" quantity="0"  label="0–5m"/>
      <ColorMapEntry color="#fd8d3c" quantity="15" label="5–15m"/>
      <ColorMapEntry color="#bd0026" quantity="50" label="15–50m"/>
    </ColorMap>
  </RasterSymbolizer>
</Rule>
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

Which GeoServer service returns **raw vector features** (not rendered images) that a JavaScript client can style dynamically?

- A) WMS
- B) **WFS** ✅
- C) WCS
- D) WPS

**Question 2 — True or False**

TiTiler can serve map tiles directly from a Cloud-Optimized GeoTIFF (COG) stored in an S3 bucket or local disk without loading the entire file into memory first.

- A) **True** ✅ *(COGs use HTTP range requests — TiTiler reads only the needed tile portion)*
- B) False

**Question 3 — Scenario**

Your UDT needs to serve a 2GB Land Surface Temperature raster (updated daily from Sentinel-3) as a dynamic, browser-viewable tile layer with a custom color ramp. Which tool is best suited?

- A) GeoServer WCS
- B) PostGIS Raster
- C) **TiTiler** ✅
- D) Leaflet

---

## 📖 Part 2 — TiTiler and the Raster Tile Pipeline (~4 min)

### 5.4 TiTiler: Modern Raster Tile Serving

**TiTiler** is a modern, lightweight Python (FastAPI) server that dynamically generates map tiles from **Cloud-Optimized GeoTIFFs (COG)**. It is purpose-built for raster serving in cloud/web environments.

Key advantages over GeoServer for rasters:
- No file pre-processing — just point it at a COG and it serves tiles instantly
- Dynamic **color ramp** application via URL parameters
- **Multi-band** support for satellite imagery (RGB composites, NDVI, etc.)
- Lightweight, stateless — runs easily in Docker

### 5.5 Creating a COG from PostGIS Raster Data

The pipeline to get PostGIS raster data into TiTiler:

```bash
# 1. Export raster from PostGIS to GeoTIFF
gdal_translate \
  "PG:host=localhost dbname=twin user=postgres table=lst_raster" \
  output.tif

# 2. Convert to Cloud-Optimized GeoTIFF
rio cogeo create output.tif output_cog.tif --cog-profile deflate

# 3. Verify COG structure
rio cogeo validate output_cog.tif
```

### 5.6 TiTiler API: Key Endpoints

Once TiTiler is running (default: `http://localhost:8001`):

| Endpoint | Purpose |
|----------|---------|
| `/cog/tiles/{z}/{x}/{y}` | XYZ tile endpoint (for Mapbox/Leaflet) |
| `/cog/info` | Metadata (bands, CRS, stats) |
| `/cog/statistics` | Per-band statistics |
| `/cog/map` | Browser preview |

**Dynamic color ramp via URL:**
```
/cog/tiles/12/2100/1387?
  url=/data/lst_20240601.tif
  &colormap_name=RdYlBu_r
  &rescale=25,45
```
This renders a red-yellow-blue tile for the area tile `12/2100/1387`, rescaling from 25°C (cool) to 45°C (hot) — perfect for urban heat maps.

### 5.7 Integrating GeoServer + TiTiler with Mapbox GL JS

```javascript
// Add a WFS vector layer from GeoServer
map.addSource('buildings', {
  type: 'geojson',
  data: 'http://geoserver:8080/geoserver/ows?service=WFS&version=2.0.0' +
        '&request=GetFeature&typeName=twin:buildings&outputFormat=application/json'
});

// Add a raster tile layer from TiTiler
map.addSource('lst-raster', {
  type: 'raster',
  tiles: ['http://titiler:8001/cog/tiles/{z}/{x}/{y}?' +
          'url=/data/lst_latest.tif&colormap_name=inferno&rescale=20,50'],
  tileSize: 256
});
```

### 5.8 GeoServer vs. TiTiler — Decision Guide

| Criterion | Use GeoServer | Use TiTiler |
|-----------|--------------|-------------|
| Data type | Vector (polygons, lines, points) | Raster (GeoTIFF, COG) |
| Protocol | OGC standards (WMS/WFS/WCS) | REST tiles (XYZ) |
| Dynamic styling | SLD styles | URL color ramp parameters |
| Update frequency | Static/daily | Near-real-time (COG swap) |
| Complexity | High config | Minimal config |

> 💡 In a production UDT, **both tools are used together** — GeoServer for vector urban layers, TiTiler for frequently updated raster outputs (satellite imagery, model results, sensor heatmaps).

---

## 🔑 Key Takeaways

1. The **serving layer** decouples storage (PostGIS) from visualization, enabling web clients to consume data via HTTP.
2. **GeoServer** is ideal for vector data and OGC-standard services (WMS, WFS, WCS).
3. **TiTiler** is purpose-built for raster tile serving from COG files — lightweight, fast, and dynamic.
4. A **Cloud-Optimized GeoTIFF (COG)** is the key intermediate format: exportable from PostGIS and consumable by TiTiler with zero preprocessing.
5. In production UDTs, GeoServer and TiTiler operate in parallel within the same Docker stack.

---

## 📎 References & Further Reading

- GeoServer Documentation: https://docs.geoserver.org/
- TiTiler Documentation: https://developmentseed.org/titiler/
- rio-cogeo: https://cogeotiff.github.io/rio-cogeo/
- Cloud Optimized GeoTIFF spec: https://cogeo.org/

---

[← L4](lecture-04.md) · [Module 2 Index](index.md) · [Next: L6 – Urban Data Sources →](lecture-06.md)
