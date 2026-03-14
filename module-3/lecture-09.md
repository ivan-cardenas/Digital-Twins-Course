---
title: "L9 – Web Visualization: Mapbox GL JS & deck.gl"
layout: default
---

# Lecture 9 — Web Visualization: Mapbox GL JS & deck.gl

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 3 · Lecture 9 of 18**  
> 🎯 **Objective:** Build 2D/3D web map interfaces for a UDT using Mapbox GL JS and deck.gl

---

## 🎬 Lecture Overview

In this lecture you will learn:
- Mapbox GL JS: vector tile rendering and layer control
- deck.gl: GPU-accelerated large-dataset visualization
- When to use each tool (and when to use both together)
- Building an interactive UDT dashboard layer

---

## 📖 Part 1 — Mapbox GL JS for 2D/2.5D Visualization (~4 min)

### 9.1 Mapbox GL JS Overview

**Mapbox GL JS** is an open-source JavaScript library for interactive, WebGL-powered maps. It renders **vector tiles** (MVT format) and **raster tiles** dynamically in the browser.

Key features for UDTs:
- **Vector tile rendering** — crisp at any zoom, dynamic styling
- **Extrusion** — polygon → 3D box (buildings as 2.5D blocks)
- **Paint expressions** — data-driven styling (color = function of attribute)
- **Layer stacking** — combine multiple data sources visually

### 9.2 Connecting Mapbox GL JS to a UDT Backend

```javascript
// Add building footprints from GeoServer WFS
map.addSource('buildings', {
  type: 'geojson',
  data: 'http://geoserver:8080/geoserver/ows?service=WFS&version=2.0.0' +
        '&request=GetFeature&typeName=twin:buildings&outputFormat=application/json'
});

map.addLayer({
  id: 'buildings-3d',
  type: 'fill-extrusion',
  source: 'buildings',
  paint: {
    'fill-extrusion-height': ['get', 'height_m'],
    'fill-extrusion-color': [
      'interpolate', ['linear'], ['get', 'lst_value'],
      25, '#313695',   // cool
      35, '#ffffbf',   // neutral
      45, '#d73027'    // hot
    ],
    'fill-extrusion-opacity': 0.85
  }
});

// Add LST raster from TiTiler
map.addSource('lst', {
  type: 'raster',
  tiles: ['http://titiler:8001/cog/tiles/{z}/{x}/{y}?url=/data/lst.tif&colormap_name=RdYlBu_r&rescale=25,45']
});
map.addLayer({ id: 'lst-overlay', type: 'raster', source: 'lst', paint: { 'raster-opacity': 0.6 } });
```

### 9.3 Data-Driven Styling for Urban Analysis

Mapbox GL JS expressions enable powerful **data-driven cartography**:

```javascript
// Color neighborhoods by green coverage percentage
'fill-color': [
  'step', ['get', 'green_pct'],
  '#d73027', 10,   // < 10% → red
  '#fc8d59', 20,   // 10–20% → orange
  '#fee08b', 30,   // 20–30% → yellow
  '#d9ef8b', 40,   // 30–40% → light green
  '#1a9850'        // > 40% → dark green
]
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

Which Mapbox GL JS layer type creates a **3D extruded building** visualization from polygon footprints?

- A) `fill`
- B) `line`
- C) **`fill-extrusion`** ✅
- D) `symbol`

**Question 2 — Best tool match**

You need to render **500,000 individual trip trajectories** (lines) on a map simultaneously, updating in real time. Which tool is better suited?

- A) Mapbox GL JS (standard layer)
- B) **deck.gl** ✅ *(GPU-accelerated, designed for massive datasets)*

---

## 📖 Part 2 — deck.gl for Large-Scale Urban Data (~4 min)

### 9.4 When Standard Rendering Breaks Down

Mapbox GL JS handles thousands of features well. But urban datasets can be **massive**:
- 1 million GPS tracks from a mobility study
- 500,000 individual parcel polygons with attributes
- Real-time particle simulation for air pollution dispersion

For these, **deck.gl** is the tool of choice.

### 9.5 deck.gl Architecture

**deck.gl** (by Uber / vis.gl) is a WebGL-powered framework for large-scale geospatial visualizations. It uses the **GPU** for geometry processing, enabling:

- 10–100× more features than traditional web maps
- Smooth animation of temporal data
- Built-in layers optimized for specific data types

### 9.6 Key deck.gl Layers for UDTs

| Layer | Use Case |
|-------|---------|
| `ScatterplotLayer` | Sensor locations, incidents |
| `HeatmapLayer` | Density of events (accidents, complaints) |
| `PathLayer` | Vehicle trajectories, utility routes |
| `PolygonLayer` | Parcels, zones (high-count) |
| `HexagonLayer` | Aggregated density grid |
| `TripsLayer` | Animated mobility data |
| `ColumnLayer` | 3D bar charts on map |

### 9.7 Example: Animated Mobility Data

```javascript
import DeckGL from '@deck.gl/react';
import { TripsLayer } from '@deck.gl/geo-layers';

<DeckGL
  viewState={{ longitude: 6.9, latitude: 52.22, zoom: 13, pitch: 45 }}
  layers={[
    new TripsLayer({
      id: 'trips',
      data: mobilityTrips,         // [{path: [[lon,lat,time],...]}]
      getPath: d => d.path,
      getTimestamps: d => d.timestamps,
      getColor: [253, 128, 93],
      currentTime: animationTime,  // update from setInterval
      trailLength: 200,
      widthMinPixels: 2
    })
  ]}
/>
```

### 9.8 Combining Mapbox GL JS + deck.gl

The two libraries work **together** in the same application:

```javascript
// deck.gl renders on top of a Mapbox GL JS basemap
import DeckGL from '@deck.gl/react';
import Map from 'react-map-gl';

<DeckGL layers={[deckLayer]} viewState={viewState}>
  <Map mapStyle="mapbox://styles/mapbox/dark-v11" />
</DeckGL>
```

This pattern is common in production UDT dashboards:
- **Mapbox GL JS** provides: basemap, labeled streets, building extrusions from WFS
- **deck.gl** provides: sensor overlays, mobility animations, density heatmaps

---

## 🔑 Key Takeaways

1. **Mapbox GL JS** excels at styled, interactive 2D/2.5D maps connected to GeoServer/TiTiler backends.
2. **fill-extrusion** layers transform polygon footprints into 3D buildings driven by attribute data.
3. **deck.gl** handles massive datasets (100k–1M features) through GPU acceleration.
4. In production UDTs, **both libraries are combined**: Mapbox GL JS for the base map and vector layers, deck.gl for large-scale overlays and animated data.

---

## 📎 References & Further Reading

- Mapbox GL JS: https://docs.mapbox.com/mapbox-gl-js/
- deck.gl: https://deck.gl/
- react-map-gl: https://visgl.github.io/react-map-gl/
- vis.gl layers catalog: https://deck.gl/docs/api-reference/layers

---

[← L8](lecture-08.md) · [Module 3 Index](index.md) · [Next: Module 4 →](../module-4/index.md)
