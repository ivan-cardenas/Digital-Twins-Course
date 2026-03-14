---
title: "L8 – 3D Tiles & Cesium"
layout: default
---

# Lecture 8 — 3D Tiles & Cesium: Streaming City-Scale 3D

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 3 · Lecture 8 of 18**  
> 🎯 **Objective:** Stream massive 3D city models to the browser using the 3D Tiles specification and CesiumJS

---

## 🎬 Lecture Overview

In this lecture you will learn:
- The 3D Tiles specification and why it was created
- How hierarchical tiling enables city-scale 3D in the browser
- CesiumJS: the open-source 3D globe rendering engine
- Building a 3D UDT viewer with real urban data

---

## 📖 Part 1 — The 3D Tiles Specification (~4 min)

### 8.1 The Problem: City-Scale 3D is Huge

A LoD2 model of a city of 200,000 buildings might contain:
- **500 million triangles**
- **50 GB of data**

No browser can load this at once. The solution: **stream only what the viewer needs, when they need it.**

### 8.2 3D Tiles: Hierarchical Spatial Tiling

**3D Tiles** is an OGC community standard (developed by Cesium) that organizes 3D content into a **hierarchical spatial data structure** (a bounding-volume hierarchy).

The structure:
```
tileset.json  ← root: describes hierarchy + bounding volumes
├── tile_0_0.b3dm   ← city-wide low-resolution model
├── tile_1_0.b3dm   ← district-level medium detail
│   ├── tile_2_0.b3dm  ← block-level
│   │   └── tile_3_0.b3dm  ← building-level (full LoD2)
```

**The renderer only fetches tiles that are:**
- Within the camera's view frustum
- At an appropriate resolution for the current zoom level

This enables rendering cities of millions of buildings at **60 fps** on a standard laptop.

### 8.3 3D Tile Formats

| Format | Content | Use |
|--------|---------|-----|
| **b3dm** (Batched 3D Model) | Batched glTF buildings | Building models |
| **i3dm** (Instanced 3D Model) | Repeated instances (trees) | Vegetation |
| **pnts** (Point Cloud) | LiDAR point clouds | Terrain, dense scans |
| **cmpt** (Composite) | Mixed tile types | Complex scenes |
| **glTF 2.0** (implicit) | Modern 3D model format | Next generation |

### 8.4 Converting CityGML to 3D Tiles

```bash
# Using 3DCityDB Exporter + CesiumJS plugin
java -jar impexp.jar export \
  -H localhost -d citydb \
  -t Building \
  --cesium-option lod=2 \
  -o ./output/tileset.json

# Or using py3dtilers (Python)
pip install py3dtilers
citygml-tiler -i city_model.gml -o ./tiles/
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

What is the primary purpose of 3D Tiles' hierarchical bounding-volume structure?

- A) To improve color rendering quality
- B) **To enable streaming — loading only visible, appropriately detailed tiles** ✅
- C) To store semantic building attributes
- D) To compress geometry for storage

**Question 2 — True or False**

In 3D Tiles, the browser always downloads the full city model before rendering begins.

- A) True
- B) **False** ✅ *(3D Tiles stream progressively — only visible tiles at current zoom level are fetched)*

---

## 📖 Part 2 — CesiumJS and the 3D UDT Viewer (~4 min)

### 8.5 CesiumJS: Open-Source 3D Globe

**CesiumJS** is a JavaScript library for rendering 3D geospatial data — including globe, terrain, imagery, and 3D Tiles — using WebGL.

**Key features for UDTs:**
- 3D Tiles streaming (buildings, point clouds, BIM)
- Terrain streaming (Cesium World Terrain or custom DTM)
- Time-dynamic visualization (animate sensor data over time)
- Clipping planes (section cuts through buildings)
- Shadow simulation (solar analysis)

### 8.6 Building a Basic 3D City Viewer

```javascript
import { Viewer, Cesium3DTileset, Ion } from 'cesium';
import 'cesium/Build/Cesium/Widgets/widgets.css';

const viewer = new Viewer('cesiumContainer', {
  terrain: Cesium.Terrain.fromWorldTerrain(),
});

// Load local 3D Tiles tileset
const tileset = await Cesium3DTileset.fromUrl(
  'http://localhost:8080/tiles/tileset.json'
);
viewer.scene.primitives.add(tileset);
viewer.zoomTo(tileset);

// Style buildings by height (color gradient)
tileset.style = new Cesium.Cesium3DTileStyle({
  color: {
    conditions: [
      ['${height} >= 50', 'color("red")'],
      ['${height} >= 20', 'color("orange")'],
      ['true',            'color("white")']
    ]
  }
});
```

### 8.7 Adding Time-Dynamic IoT Data

```javascript
// Animate air quality sensor readings over time
const dataSource = new Cesium.CustomDataSource('sensors');

sensors.forEach(sensor => {
  dataSource.entities.add({
    position: Cesium.Cartesian3.fromDegrees(sensor.lon, sensor.lat),
    point: {
      pixelSize: 12,
      color: new Cesium.CallbackProperty(() => {
        const val = getCurrentAQI(sensor.id);
        return val > 100 ? Cesium.Color.RED : Cesium.Color.GREEN;
      }, false)
    }
  });
});

viewer.dataSources.add(dataSource);
```

---

## 🔑 Key Takeaways

1. **3D Tiles** solves the city-scale rendering problem through hierarchical streaming — only necessary detail is loaded.
2. **b3dm** is the primary tile format for buildings; **pnts** handles LiDAR point clouds.
3. **CesiumJS** provides a full-featured WebGL globe renderer that natively consumes 3D Tiles.
4. Dynamic styling and time-based data make Cesium a powerful platform for UDT visualization including sensor overlays.

---

## 📎 References & Further Reading

- 3D Tiles Specification: https://cesium.com/why-cesium/3d-tiles/
- CesiumJS Documentation: https://cesium.com/learn/cesiumjs/
- py3dtilers: https://github.com/VCityTeam/py3dtilers
- 3DCityDB Web Map Client: https://www.3dcitydb.org/

---

[← L7](lecture-07.md) · [Module 3 Index](index.md) · [Next: L9 – Mapbox GL JS & deck.gl →](lecture-09.md)
