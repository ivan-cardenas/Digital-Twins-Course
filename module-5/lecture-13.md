---
title: "L13 – DAG-Based Cross-Sectoral Methodology"
layout: default
---

# Lecture 13 — DAG-Based Cross-Sectoral Methodology

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 5 · Lecture 13 of 18**  
> 🎯 **Objective:** Formalize planning questions as Directed Acyclic Graphs (DAGs) that drive a UDT's data schema and analysis pipeline

---

## 🎬 Lecture Overview

In this lecture you will learn:
- What a Directed Acyclic Graph (DAG) is and why it fits urban analysis
- How to translate a planning question into a DAG of spatial indicators
- How DAGs map to relational database schemas (Django ORM)
- An applied cross-sectoral example: water supply + urban heat

---

## 📖 Part 1 — From Planning Questions to DAGs (~4 min)

### 13.1 The Problem with Siloed Urban Analysis

Traditional urban analysis is **sector-by-sector**:
- The water department analyses network capacity
- The urban planning department models densification
- The public health department tracks heat mortality

Each uses different tools, different datasets, different spatial units. No one sees the cross-sectoral picture:

> *"If we densify the southern district, how does that affect water demand, surface temperature, and heat-related health risk — simultaneously?"*

This kind of question requires a **cross-sectoral, causally structured framework**. That framework is the DAG.

### 13.2 What Is a DAG?

A **Directed Acyclic Graph (DAG)** is a graph where:
- **Nodes** represent variables or indicators
- **Directed edges** represent causal or computational dependencies
- **Acyclic** means there are no feedback loops (no circular causation)

```
Population Density ──→ Water Demand ──→ Supply Deficit
        │
        └──→ Impervious Surface ──→ Surface Temperature ──→ Heat Mortality
                      │
                      └──→ Stormwater Runoff ──→ Flood Risk
```

Each node is a **spatial indicator** (a map layer, a PostGIS column, or a raster dataset). Each edge is a **computation or model** that transforms one indicator into another.

### 13.3 Planning Questions as DAG Templates

A UDT can store **reusable DAG templates** for common planning questions:

| Planning Question | Root Node | Terminal Nodes |
|-------------------|-----------|---------------|
| "Impact of densification on water supply" | Population Density | Supply Deficit |
| "Effect of green roofs on urban heat" | Green Coverage % | Heat Mortality Risk |
| "Flood risk under climate change" | Precipitation Intensity | Flood Damage Index |

Each template becomes a **query plan** — the system knows which layers to load, which computations to run, and in what order.

### 13.4 DAGs and DPSIR Alignment

The DAG structure maps naturally onto DPSIR (from Lecture 3):

```
[D] Population Growth
        ↓
[P] Increased Impervious Surface
        ↓
[S] Higher Land Surface Temperature
        ↓
[I] Heat Mortality Rate
        ↓
[R] Green Infrastructure Investment  ←── policy lever
```

Each DPSIR arrow is a DAG edge. The planner can "intervene" at any node (e.g., add green infrastructure) and propagate the effect forward through the graph.

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

In a DAG representing urban causation, what do the **edges** (arrows) between nodes represent?

- A) Administrative boundaries between departments
- B) **Causal or computational dependencies between indicators** ✅
- C) Geographic distances between sensor locations
- D) Temporal gaps between data collection events

**Question 2 — True or False**

A DAG can contain cycles — for example, "Population Density → Heat Stress → Out-migration → Population Density."

- A) True
- B) **False** ✅ *(by definition, a DAG is acyclic — no cycles. Feedback loops require different modeling approaches, e.g., System Dynamics)*

**Question 3 — Apply it**

A city planner wants to assess how a new industrial zone affects air quality and respiratory health. What are the likely **root node** and **terminal node** in the corresponding DAG?

> ✅ **Expected answer:** Root node = Industrial Land Use Change (or emissions source); Terminal node = Respiratory Health Index (or hospitalization rate)

---

## 📖 Part 2 — DAGs as Database Schema Drivers (~4 min)

### 13.5 From DAG to Django ORM Schema

A key methodological contribution of frameworks like **CrossTwin** (University of Twente) is that planning questions formalized as DAGs can **automatically drive the database schema**.

Each DAG node becomes a Django model (PostGIS table):

```python
# Django models.py — auto-generated from DAG definition

from django.contrib.gis.db import models

class PopulationDensity(models.Model):
    """DAG Node: Driving Force"""
    neighborhood = models.ForeignKey('Neighborhood', on_delete=models.CASCADE)
    value = models.FloatField()          # persons/km²
    year = models.IntegerField()
    geom = models.MultiPolygonField()

class ImperviousSurface(models.Model):
    """DAG Node: Pressure — computed from PopulationDensity"""
    neighborhood = models.ForeignKey('Neighborhood', on_delete=models.CASCADE)
    pct_impervious = models.FloatField()  # % of area
    derived_from = models.ForeignKey(PopulationDensity, on_delete=models.SET_NULL, null=True)
    geom = models.MultiPolygonField()

class LandSurfaceTemperature(models.Model):
    """DAG Node: State — derived from ImperviousSurface + EO data"""
    neighborhood = models.ForeignKey('Neighborhood', on_delete=models.CASCADE)
    lst_mean_c = models.FloatField()     # °C
    raster_source = models.CharField(max_length=200)
    geom = models.MultiPolygonField()
```

### 13.6 Schema-Aware Data Ingestion

Once the DAG is formalized and the schema exists, the UDT's **ingestion interface** can be schema-aware:

- It knows which layers are needed for each DAG node
- It validates incoming data against the expected geometry type, CRS, and attribute names
- It automatically computes downstream nodes when upstream data is updated

```python
# Ingestion pipeline triggered by new Sentinel-2 imagery
def on_new_sentinel_image(raster_path):
    # Compute NDVI and impervious surface
    ndvi = compute_ndvi(raster_path)
    impervious = compute_impervious_pct(ndvi)
    
    # Store to PostGIS (ImperviousSurface node)
    ImperviousSurface.objects.update_or_create(
        neighborhood=nb,
        defaults={'pct_impervious': impervious, 'geom': nb.geom}
    )
    
    # Trigger downstream DAG nodes
    update_lst_estimates()
    update_heat_risk_scores()
```

### 13.7 Cross-Sectoral Analysis: Water + Heat Example

A single DAG can span multiple sectors simultaneously:

```
          ┌── Water Demand ──→ Supply Deficit ──→ Water Stress Index
          │
Pop. Density
          │
          └── Imperv. Surface ──→ LST ──→ Heat Risk ──→ Vulnerable Pop. Exposed
                    │
                    └── Runoff Coefficient ──→ Peak Stormwater ──→ Flood Risk
```

This graph can be:
- **Visualized** as a layer-switching dashboard
- **Queried** spatially ("which neighborhoods have HIGH heat risk AND HIGH water stress?")
- **Simulated** by changing one node and propagating the effect

```sql
-- Cross-sectoral query: neighborhoods with combined risk
SELECT
    n.name,
    wr.water_stress_index,
    hr.heat_risk_score,
    (wr.water_stress_index + hr.heat_risk_score) / 2 AS combined_risk
FROM neighborhoods n
JOIN water_stress wr ON wr.neighborhood_id = n.id
JOIN heat_risk hr ON hr.neighborhood_id = n.id
WHERE wr.water_stress_index > 0.6 AND hr.heat_risk_score > 0.7
ORDER BY combined_risk DESC;
```

---

## 🔑 Key Takeaways

1. **DAGs** formalize planning questions as networks of spatial indicators connected by causal dependencies.
2. DAG nodes map naturally to **DPSIR components** — they can be interpreted as Driving Forces, Pressures, States, Impacts, or Responses.
3. A DAG-to-schema approach (as in CrossTwin) allows planning questions to **automatically drive** the database structure and ingestion pipeline.
4. Cross-sectoral queries across multiple DAG branches (water + heat + flood) are straightforward SQL joins once all indicators share a **common spatial unit** (neighborhood, parcel).

---

## 📎 References & Further Reading

- Pearl, J. (2009). *Causality: Models, Reasoning, and Inference*. Cambridge University Press.
- Cárdenas-León, I., et al. (2025). CrossTwin: A Cross-Sectoral Urban Digital Twin. *ISPRS Annals*.
- Kourtit, K., et al. (2021). Urban digital twins and planning support. *Environment and Planning B*.

---

[← L12](../module-4/lecture-12.md) · [Module 5 Index](index.md) · [Next: L14 – Scenario Analysis →](lecture-14.md)
