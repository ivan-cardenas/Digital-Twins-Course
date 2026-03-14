---
title: "L7 – CityGML & Levels of Detail (LoD)"
layout: default
---

# Lecture 7 — CityGML & Levels of Detail (LoD)

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 3 · Lecture 7 of 18**  
> 🎯 **Objective:** Understand the CityGML data model and how LoD levels determine UDT capability

---

## 🎬 Lecture Overview

In this lecture you will learn:
- What CityGML is and why it matters for Urban Digital Twins
- The five Levels of Detail (LoD 0–4) and their trade-offs
- CityGML thematic modules relevant to urban planning
- How to import CityGML data into PostGIS using 3DCityDB

---

## 📖 Part 1 — The CityGML Standard (~4 min)

### 7.1 What Is CityGML?

**CityGML** (City Geography Markup Language) is an **OGC open standard** for the representation, storage, and exchange of **semantic 3D city models**. It extends GML (Geography Markup Language) with city-specific object classes.

Key word: **semantic**. CityGML does not just store geometry — it stores geometry *with meaning*:

```
Building:
  - id: BLD_001
  - function: Residential
  - yearOfConstruction: 1985
  - storeyCount: 4
  - geometry (LoD2): roof + wall surfaces
  - address: {...}
  - boundarySurface: [RoofSurface, WallSurface, GroundSurface]
```

This semantic richness is what makes CityGML-based models suitable for **simulation and analysis** — not just visualization.

### 7.2 Thematic Modules in CityGML 3.0

CityGML organizes the city into thematic modules:

| Module | Covers |
|--------|--------|
| **Building** | Structures, facades, rooms, openings |
| **Bridge** | Bridges with geometric detail |
| **Transportation** | Roads, tracks, railways, footpaths |
| **Vegetation** | Trees, plant covers, agricultural areas |
| **WaterBody** | Rivers, lakes, canals |
| **LandUse** | Zoning, parcel use classification |
| **Relief** | Terrain surface (DTM/DEM) |
| **CityFurniture** | Street lights, benches, signs |
| **Tunnel** | Underground infrastructure |

### 7.3 The Five Levels of Detail

LoD defines the **geometric and semantic resolution** of a city model:

| LoD | Geometry | Semantics | Typical Source | Use Case |
|-----|---------|-----------|---------------|---------|
| **LoD 0** | Footprint only (2D) | Minimal | Cadastre | Regional planning |
| **LoD 1** | Block extrusion (flat roof) | Building height | Cadastre + LiDAR | City-wide shadow/heat |
| **LoD 2** | Roof shape differentiated | Roof type, walls | LiDAR + photogrammetry | Energy simulation |
| **LoD 3** | Architectural details (doors, windows) | Opening types | BIM / detailed survey | Evacuation, acoustics |
| **LoD 4** | Interior rooms modeled | Room use, furniture | BIM interior | Emergency response, navigation |

> 💡 Most city-scale UDTs operate at **LoD 1–2**. LoD 3–4 is used for critical infrastructure or landmark buildings where detailed simulation is needed.

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

You need to run a city-wide solar irradiance simulation to identify buildings suitable for rooftop solar panels. The simulation requires roof geometry (pitch, orientation) but not interior details. Which LoD is the minimum appropriate level?

- A) LoD 0
- B) LoD 1
- C) **LoD 2** ✅
- D) LoD 4

**Question 2 — Fill in the blank**

CityGML is described as a **semantic** 3D city model standard because it stores geometry ___________.

- A) in binary format
- B) **together with meaning/attributes about what each element is** ✅
- C) at very high resolution
- D) using raster pixels instead of vectors

**Question 3 — Match**

| LoD | Geometry Description |
|-----|---------------------|
| LoD 0 | Building footprint (2D polygon) |
| LoD 1 | Simple block extrusion with flat roof |
| LoD 2 | Differentiated roof shape (gable, hip, flat) |
| LoD 3 | Architectural features: windows, doors, balconies |

> ✅ All correct as shown.

---

## 📖 Part 2 — Working with CityGML in a UDT (~4 min)

### 7.4 3DCityDB: Storing CityGML in PostGIS

**3DCityDB** (3D City Database) is an open-source PostGIS schema designed to store CityGML data. It provides:

- A relational schema that maps all CityGML object types
- Import/export tools (Importer/Exporter GUI + CLI)
- WFS plugin for OGC-compliant data serving
- 3D web client for visualization

**Importing a CityGML file:**

```bash
# Using the 3DCityDB CLI importer
java -jar impexp.jar import \
  -H localhost -P 5432 -d citydb -u postgres \
  enschede_lod2.gml
```

### 7.5 Querying Semantic 3D Data with PostGIS

Once imported, CityGML data can be queried spatially and semantically:

```sql
-- Find all residential buildings over 4 floors within a district
SELECT
    b.gmlid,
    b.storeycount,
    ST_Area(b.lod2_solid::geometry) AS footprint_area
FROM building b
JOIN cityobject co ON b.id = co.id
JOIN thematic_surface ts ON ts.building_id = b.id
WHERE
    b.function = '1000'              -- residential code
    AND b.storeycount > 4
    AND ST_Within(
        co.envelope,
        (SELECT geom FROM districts WHERE name = 'Roombeek')
    );
```

### 7.6 CityGML 3.0 and IFC/BIM Integration

**CityGML 3.0** (2021) introduced formal alignment with **IFC** (Industry Foundation Classes — the BIM standard), enabling:

- Importing detailed building models from BIM authoring tools (Revit, ArchiCAD)
- Populating LoD 3–4 models from construction drawings
- Linking construction metadata (materials, energy performance) to spatial objects

This **BIM-to-GIS pipeline** is increasingly important for:
- New development projects (accurate from construction documents)
- Building energy modeling (material properties + geometry)
- Facility management integration

**Tool chain:**
```
Revit / ArchiCAD (BIM)
        ↓ IFC export
    FME / hale studio
        ↓ IFC → CityGML conversion
    3DCityDB
        ↓ PostGIS storage
    TiTiler / GeoServer
        ↓ Web serving
```

---

## 🔑 Key Takeaways

1. **CityGML** is an OGC standard for semantic 3D city models — it stores geometry *and* meaning.
2. **Levels of Detail (LoD 0–4)** define the resolution trade-off: higher LoD = more accurate simulation, higher cost.
3. Most city-scale UDTs use **LoD 1–2** for coverage, reserving LoD 3–4 for critical buildings.
4. **3DCityDB** provides a ready-made PostGIS schema for storing CityGML data and querying it spatially.
5. **CityGML 3.0** enables formal alignment with IFC/BIM, unlocking high-fidelity data from construction workflows.

---

## 📎 References & Further Reading

- OGC CityGML 3.0 Standard: https://www.ogc.org/standards/citygml
- 3DCityDB: https://www.3dcitydb.org/
- Biljecki, F., et al. (2016). An improved LoD specification for 3D building models. *Computers, Environment and Urban Systems*, 59, 25–37.
- hale studio (IFC/CityGML conversion): https://www.wetransform.to/

---

[← L6](../module-2/lecture-06.md) · [Module 3 Index](index.md) · [Next: L8 – 3D Tiles & Cesium →](lecture-08.md)
