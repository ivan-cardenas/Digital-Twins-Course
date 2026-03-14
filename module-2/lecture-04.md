---
title: "L4 – Spatial Databases: PostGIS in Depth"
layout: default
---

# Lecture 4 — Spatial Databases: PostGIS in Depth

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 2 · Lecture 4 of 18**  
> 🎯 **Objective:** Understand how PostGIS stores, indexes, and queries spatial data for urban applications

---

## 🎬 Lecture Overview

In this lecture you will learn:
- Why relational databases need spatial extensions
- PostGIS geometry types relevant to urban data
- Key spatial SQL functions for UDT queries
- How spatial indexes (GiST) make queries fast at city scale

---

## 📖 Part 1 — Why PostGIS? (~4 min)

### 4.1 The Problem with Standard Databases

Standard SQL databases (MySQL, SQLite) treat location data as plain numbers — two columns for latitude and longitude. This means:

- ❌ No native geometry types (polygons, multilines)
- ❌ No spatial operations (intersection, buffer, union)
- ❌ No spatial indexing — every location query is a full table scan
- ❌ No coordinate system management (EPSG codes)

For a city with millions of parcels, buildings, pipes, and sensor points, this is a fundamental barrier.

### 4.2 PostGIS: Spatial Extension for PostgreSQL

**PostGIS** adds to PostgreSQL:
- **Geometry and Geography types** — stores points, lines, polygons, multipolygons, 3D solids
- **Hundreds of spatial functions** — `ST_Intersects`, `ST_Buffer`, `ST_Distance`, `ST_Area`, `ST_Union`…
- **GiST spatial index** — R-tree based index for fast bounding-box queries
- **Coordinate Reference System (CRS) management** — spatial_ref_sys table with 4000+ EPSG codes
- **Raster support** (PostGIS Raster) — store and query raster grids alongside vector data

### 4.3 Geometry Types Used in Urban Digital Twins

| Geometry Type | Urban Use Case |
|--------------|----------------|
| `POINT` | Sensors, trees, manholes, bus stops |
| `LINESTRING` | Roads, rivers, pipelines, transit routes |
| `POLYGON` | Building footprints, land parcels, administrative zones |
| `MULTIPOLYGON` | Complex parcel groups, neighborhood boundaries |
| `GEOMETRYCOLLECTION` | Mixed-type datasets |
| `POLYHEDRALSURFACE` / `TIN` | 3D building models, terrain surfaces |

### 4.4 Storing an Urban Dataset in PostGIS

A buildings table for a UDT might look like:

```sql
CREATE TABLE buildings (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(200),
    use_type      VARCHAR(50),      -- residential, commercial, industrial
    floors        INTEGER,
    year_built    INTEGER,
    height_m      FLOAT,
    geom          GEOMETRY(MultiPolygon, 4326)  -- WGS84
);

-- Spatial index
CREATE INDEX buildings_geom_idx ON buildings USING GIST (geom);
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

Which spatial index type does PostGIS use for efficient bounding-box spatial queries?

- A) B-tree
- B) Hash
- C) **GiST** ✅
- D) BRIN

**Question 2 — Match the function**

Match each PostGIS function to its purpose:

| Function | Purpose |
|----------|---------|
| `ST_Buffer` | Creates a zone of a given distance around a geometry |
| `ST_Intersects` | Returns TRUE if two geometries share any space |
| `ST_Area` | Returns the area of a polygon |
| `ST_Distance` | Returns the minimum distance between two geometries |

> ✅ All correct as shown above.

**Question 3 — Code reading**

What does `GEOMETRY(MultiPolygon, 4326)` mean in a CREATE TABLE statement?

- A) A single polygon stored in EPSG:4326 (WGS84)
- B) **A multi-polygon geometry stored in WGS84 lat/lon coordinates** ✅
- C) A raster layer in UTM projection
- D) A 3D surface model

---

## 📖 Part 2 — Spatial SQL for Urban Analysis (~4 min)

### 4.5 Core Spatial Queries for UDTs

**Find all buildings within 500m of a flood zone:**

```sql
SELECT b.id, b.name, b.use_type
FROM buildings b
JOIN flood_zones fz ON ST_DWithin(b.geom, fz.geom, 500)
WHERE fz.risk_level = 'high';
```

**Calculate green coverage per neighborhood:**

```sql
SELECT
    n.name AS neighborhood,
    SUM(ST_Area(ST_Intersection(g.geom, n.geom))) /
    ST_Area(n.geom) * 100 AS green_coverage_pct
FROM neighborhoods n
JOIN green_spaces g ON ST_Intersects(g.geom, n.geom)
GROUP BY n.name, n.geom;
```

**Buffer a water pipeline and count at-risk households:**

```sql
SELECT COUNT(h.id) AS households_at_risk
FROM households h
WHERE ST_Within(
    h.geom,
    ST_Buffer(
        (SELECT geom FROM water_mains WHERE id = 42),
        0.01  -- degrees (~1km in mid-latitudes)
    )
);
```

### 4.6 Working with Rasters in PostGIS

PostGIS Raster stores satellite imagery, DEMs, or model outputs (e.g., LST from Sentinel-3) directly in the database:

```sql
-- Get average Land Surface Temperature per neighborhood
SELECT
    n.name,
    ST_SummaryStats(
        ST_Clip(r.rast, n.geom)
    ) AS lst_stats
FROM neighborhoods n
JOIN lst_raster r ON ST_Intersects(r.rast::geometry, n.geom);
```

This is powerful because **vector and raster analysis happen in the same query** — no need to export and re-import between GIS desktop tools.

### 4.7 CRS and Projections

A critical issue in multi-source UDTs: **data arrives in different coordinate systems.**

- Cadastral data: often national projection (e.g., Amersfoort / EPSG:28992 in the Netherlands)
- GPS / smartphone data: WGS84 / EPSG:4326
- Metric operations (area, distance): need a projected CRS (e.g., UTM zone)

PostGIS handles this with `ST_Transform`:

```sql
-- Convert building footprints from WGS84 to Dutch RD New for metric calculations
SELECT id, ST_Area(ST_Transform(geom, 28992)) AS area_m2
FROM buildings;
```

> 💡 **Best practice:** Store all geometries in a single CRS (project-wide), apply `ST_Transform` only in queries that need it. Most UDT implementations use EPSG:4326 for storage and EPSG:3857 (Web Mercator) for tile serving.

---

## 🔑 Key Takeaways

1. PostGIS adds **geometry types, spatial functions, and spatial indexing** to PostgreSQL — making it suitable for city-scale data.
2. A **GiST spatial index** is essential for query performance — always create one on geometry columns.
3. Core spatial queries (`ST_Buffer`, `ST_Intersects`, `ST_DWithin`, `ST_Clip`) power most UDT analytical tasks.
4. Storing **rasters and vectors in the same database** enables powerful combined analyses without file export.
5. Always manage **coordinate reference systems explicitly** — mismatched CRS is the most common source of spatial errors.

---

## 📎 References & Further Reading

- PostGIS Documentation: https://postgis.net/docs/
- Regina Obe & Leo Hsu (2021). *PostGIS in Action, 3rd Edition*. Manning.
- EPSG Registry: https://epsg.io/

---

[← L3](../module-1/lecture-03.md) · [Module 2 Index](index.md) · [Next: L5 – GeoServer & TiTiler →](lecture-05.md)
