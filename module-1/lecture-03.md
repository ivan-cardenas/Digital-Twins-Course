---
title: "L3 – The DPSIR Framework as an Analytical Lens"
layout: default
---

# Lecture 3 — The DPSIR Framework as an Analytical Lens

> 🕐 **Estimated duration:** 7–9 min  
> 📍 **Module 1 · Lecture 3 of 18**  
> 🎯 **Objective:** Apply the DPSIR framework to structure multi-sectoral urban analysis within a UDT

---

## 🎬 Lecture Overview

In this lecture you will learn:
- What the DPSIR framework is and where it comes from
- How to map urban problems onto the DPSIR chain
- Why DPSIR is a natural fit for Urban Digital Twins
- A worked example: urban heat stress

---

## 📖 Part 1 — Understanding DPSIR (~4 min)

### 3.1 What Is DPSIR?

**DPSIR** stands for **Driving forces → Pressures → State → Impact → Response**. It is a causal-chain framework developed by the **European Environment Agency (EEA)** for environmental reporting, and widely adopted in sustainability science.

```
  DRIVING FORCES
  (socio-economic)
        ↓
   PRESSURES
  (emissions, use)
        ↓
     STATE
  (env. condition)
        ↓
    IMPACTS
  (on humans/ecosys)
        ↓
   RESPONSES
  (policies, tech)
        ↑_______________|  (feedback loop)
```

### 3.2 The Five Components Defined

| Component | Definition | Urban Example |
|-----------|-----------|---------------|
| **Driving Forces (D)** | Underlying socio-economic or demographic processes | Population growth, urban sprawl, car dependency |
| **Pressures (P)** | Direct stressors on the environment caused by D | Increased impervious surfaces, GHG emissions |
| **State (S)** | Current condition of the environment/system | Air temperature, green coverage %, flood risk index |
| **Impact (I)** | Effects of the state on people, ecosystems, economy | Heat-related illness, biodiversity loss, property damage |
| **Response (R)** | Policy, management, or technical interventions | Green roofs, early warning systems, zoning changes |

### 3.3 Why DPSIR for Urban Digital Twins?

A UDT must answer questions that cross departmental boundaries:
- How does **urban densification** (D) affect **surface runoff** (P) and **flood risk** (S/I)?
- How do **green infrastructure investments** (R) reduce **urban heat** (P/S) and **health costs** (I)?

DPSIR provides:
1. A **shared vocabulary** across departments (water, urban planning, public health, transport)
2. A **directed analytical structure** — causal chains can be encoded as **DAGs (Directed Acyclic Graphs)**
3. **Indicator framework** — each component maps to measurable, spatial KPIs

> 💡 A DAG-based implementation of DPSIR is the backbone of research platforms like **CrossTwin** (University of Twente), which structures cross-sectoral urban analysis as a network of spatial indicators connected by causal dependencies.

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Fill in the blank**

In DPSIR, _______ represents the current measurable condition of the environment or urban system (e.g., the current average surface temperature of a neighborhood).

- A) Driving Force
- B) Pressure
- C) **State** ✅
- D) Impact

**Question 2 — Scenario matching**

A city installs 500 green roofs to reduce urban heat island effect. Which DPSIR component does this action represent?

- A) Driving Force
- B) Pressure
- C) Impact
- D) **Response** ✅

**Question 3 — True or False**

DPSIR can be implemented as a Directed Acyclic Graph (DAG) where nodes are spatial indicators and edges are causal relationships.

- A) **True** ✅
- B) False

---

## 📖 Part 2 — Applying DPSIR to Urban Heat Stress (~4 min)

### 3.4 Worked Example: Urban Heat Stress

Let's walk through a complete DPSIR chain for **urban heat stress** — one of the most pressing urban climate challenges.

**D — Driving Forces**
- Rapid urbanization (population growth + housing demand)
- Economic development increasing energy use and vehicles
- Climate change as a macro-level driver

**P — Pressures**
- Replacement of vegetation with impervious surfaces (concrete, asphalt)
- Waste heat from buildings and vehicles
- Reduction of albedo (dark rooftops absorbing solar radiation)

**S — State** *(spatial indicators you can map in a UDT)*
- Land Surface Temperature (LST) from **Sentinel-3 / Landsat** imagery
- Normalized Difference Vegetation Index (NDVI) from **Sentinel-2**
- % impervious surface per neighborhood (derived from aerial imagery or OSM)
- Air temperature from IoT sensor networks

**I — Impacts** *(spatial indicators mapped to people)*
- Heat-related mortality and hospitalization rates (linked to neighborhood boundaries)
- Outdoor thermal comfort index (Universal Thermal Climate Index — UTCI)
- Economic losses (reduced labor productivity, increased cooling energy costs)
- Disproportionate impact on vulnerable populations (elderly, low-income)

**R — Responses** *(policy + technology)*
- Urban greening programs (parks, street trees, green roofs)
- Cool pavement / reflective roof mandates
- Heat early warning systems (fed by live LST data from the UDT)
- Zoning revisions requiring green coverage ratios

### 3.5 DPSIR as a Data Model

Each DPSIR component translates to **geospatial layers** in the UDT:

```
Driving Forces → Census data, land use change maps, EO time series
Pressures      → Impervious surface rasters, emissions point data
State          → LST rasters, NDVI layers, sensor readings
Impacts        → Health statistics per tract, flood damage polygons
Responses      → Green infrastructure polygons, policy zones
```

And the **causal edges** between them can be stored as a graph structure (DAG), enabling:
- Traceability: which indicator caused which impact?
- Scenario simulation: if I change R, what happens to S and I?
- Multi-sector reporting: same framework applied to water, housing, and heat simultaneously

### 3.6 DPSIR in Practice: Multi-Sector Application

| Sector | D | P | S | I | R |
|--------|---|---|---|---|---|
| **Water Supply** | Pop. growth | Over-extraction | Groundwater level | Water scarcity | Demand management |
| **Urban Heat** | Urbanization | Imperviousness | LST / NDVI | Heat mortality | Greening programs |
| **Housing** | Migration | Housing demand | Affordability index | Displacement | Social housing policy |

> This is the cross-sectoral power of DPSIR: **one framework, three sectors, one digital twin.**

---

## 🔑 Key Takeaways

1. DPSIR is a causal-chain framework: **D → P → S → I → R**, with a feedback loop from R back to D.
2. Each component maps to **geospatial, measurable indicators** that can live as layers in a UDT.
3. DPSIR enables **cross-sectoral analysis** — the same structure applies to heat, water, housing, and more.
4. Implemented as a **DAG**, DPSIR becomes a computational model for scenario analysis and impact tracing inside a digital twin.

---

## 📎 References & Further Reading

- EEA. (1999). *Environmental Indicators: Typology and Overview*. European Environment Agency.
- Smeets, E., & Weterings, R. (1999). *Environmental Indicators: Typology and Overview*. EEA Technical Report No. 25.
- Gari, S.R., et al. (2015). A review of the application and evolution of the DPSIR framework. *Journal of Cleaner Production*, 103, 784–793.
- Van Bennekom-Minnema, J., et al. (2022). Urban Digital Twins and the DPSIR Framework. *Urban Climate*.

---

[← L2](lecture-02.md) · [Module 1 Index](index.md) · [Next: Module 2 →](../module-2/index.md)
