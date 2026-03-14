---
title: "L14 – Scenario Analysis & What-If Simulation"
layout: default
---

# Lecture 14 — Scenario Analysis & What-If Simulation

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 5 · Lecture 14 of 18**  
> 🎯 **Objective:** Design and run spatial what-if scenarios within a UDT to compare policy interventions

---

## 🎬 Lecture Overview

In this lecture you will learn:
- What scenario analysis means in a UDT context
- How to implement a "baseline vs. intervention" comparison framework
- Spatial scenario tools: geospatial editing + model propagation
- A worked example: comparing two green infrastructure policies

---

## 📖 Part 1 — Scenario Analysis Framework (~4 min)

### 14.1 What Is Scenario Analysis in a UDT?

A **scenario** is a hypothetical change to one or more DAG nodes, propagated forward through the causal graph to see its effect on downstream indicators.

> *"What happens to heat risk and stormwater runoff if we convert 20% of parking lots in the city center to green spaces?"*

This requires:
1. A **baseline** — current state of all indicators
2. A **scenario definition** — the change(s) to make
3. A **propagation model** — how the change ripples through the DAG
4. A **comparison layer** — the difference between baseline and scenario

### 14.2 Three Scenario Types in Urban Planning

| Type | Description | Example |
|------|-------------|---------|
| **Trend** | Business-as-usual projection | Population grows at current rate → water demand in 2035 |
| **Policy** | A specific intervention | 500 green roofs → heat reduction |
| **Climate** | External forcing changes | +2°C air temperature → effect on LST and flood frequency |

A UDT supports all three — often combining them (e.g., policy + climate scenario simultaneously).

### 14.3 Scenario Storage Pattern in PostGIS

Scenarios are stored as **parallel indicator tables** with a `scenario_id`:

```sql
CREATE TABLE scenario (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    description TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    is_baseline BOOLEAN DEFAULT FALSE
);

CREATE TABLE land_surface_temperature_scenario (
    id          SERIAL PRIMARY KEY,
    scenario_id INTEGER REFERENCES scenario(id),
    neighborhood_id INTEGER REFERENCES neighborhoods(id),
    lst_mean_c  FLOAT,
    geom        GEOMETRY(MultiPolygon, 4326)
);
```

**Query: difference between baseline and green-roof scenario**

```sql
SELECT
    n.name,
    base.lst_mean_c AS baseline_lst,
    scen.lst_mean_c AS scenario_lst,
    base.lst_mean_c - scen.lst_mean_c AS reduction_c
FROM neighborhoods n
JOIN land_surface_temperature_scenario base
    ON base.neighborhood_id = n.id AND base.scenario_id = 1   -- baseline
JOIN land_surface_temperature_scenario scen
    ON scen.neighborhood_id = n.id AND scen.scenario_id = 5   -- green roof policy
ORDER BY reduction_c DESC;
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

In a UDT scenario analysis, what is the purpose of the **baseline** scenario?

- A) The optimal policy target
- B) **The current (or business-as-usual) state against which interventions are compared** ✅
- C) The worst-case climate projection
- D) The historical average over 10 years

**Question 2 — Scenario classification**

Classify each scenario:

| Scenario | Type |
|----------|------|
| "What if summer temperatures rise by 1.5°C by 2050?" | Climate |
| "What if we build 2,000 social housing units in the northern district?" | Policy |
| "What if the current population growth rate continues for 10 years?" | Trend |

> ✅ All correct.

**Question 3 — True or False**

A well-designed UDT can run a scenario that simultaneously models a **policy** change (green roofs) AND a **climate** change (+2°C) and compare the result to the baseline.

- A) **True** ✅
- B) False

---

## 📖 Part 2 — Running and Visualizing Scenarios (~4 min)

### 14.4 Spatial Editing as Scenario Input

The scenario starts with a **spatial edit** — a planner draws or uploads the proposed change on a map:

```javascript
// React + Mapbox GL JS: draw proposed green spaces
import MapboxDraw from '@mapbox/mapbox-gl-draw';

const draw = new MapboxDraw({
  modes: { ...MapboxDraw.modes },
  displayControlsDefault: false,
  controls: { polygon: true, trash: true }
});

map.addControl(draw);

map.on('draw.create', async (e) => {
  const drawnPolygon = e.features[0];
  // Send to Django backend to run scenario model
  await fetch('/api/scenario/run/', {
    method: 'POST',
    body: JSON.stringify({
      scenario_id: currentScenarioId,
      intervention_type: 'green_space',
      geometry: drawnPolygon.geometry
    })
  });
});
```

### 14.5 Server-Side Scenario Propagation

The backend runs the DAG propagation model:

```python
# Django view: propagate a green-space intervention through the DAG
@api_view(['POST'])
def run_scenario(request):
    scenario_id = request.data['scenario_id']
    geom = GEOSGeometry(json.dumps(request.data['geometry']))

    # Step 1: Update impervious surface (Pressure node)
    affected = Neighborhood.objects.filter(geom__intersects=geom)
    for nb in affected:
        overlap_area = nb.geom.intersection(geom).area
        reduction_pct = (overlap_area / nb.geom.area) * 100
        ImperviousSurfaceScenario.objects.update_or_create(
            scenario_id=scenario_id, neighborhood=nb,
            defaults={'pct_impervious': nb.baseline_impervious - reduction_pct}
        )

    # Step 2: Re-compute LST (State node) using regression model
    compute_lst_from_impervious(scenario_id)

    # Step 3: Re-compute heat risk (Impact node)
    compute_heat_risk(scenario_id)

    return Response({'status': 'ok', 'affected_neighborhoods': affected.count()})
```

### 14.6 Side-by-Side Visualization

The UDT dashboard renders scenarios as a **swipe comparison** or **split map**:

```jsx
// Split-map comparison: baseline (left) vs. scenario (right)
function ScenarioCompare({ baselineId, scenarioId }) {
  return (
    <div style={{ display: 'flex', height: '100vh' }}>
      <MapPanel
        title="Baseline"
        layerUrl={`/api/tiles/lst/?scenario=${baselineId}`}
        colormap="RdYlBu_r"
      />
      <MapPanel
        title="Green Roof Policy 2030"
        layerUrl={`/api/tiles/lst/?scenario=${scenarioId}`}
        colormap="RdYlBu_r"
      />
    </div>
  );
}
```

### 14.7 Communicating Results to Planners

Scenario analysis outputs must be actionable. Key outputs for planning practice:

| Output | Format | Audience |
|--------|--------|---------|
| Difference map (Δ LST) | Choropleth layer | Urban planner |
| Bar chart: reduction per district | Chart | Policymaker |
| Cost-benefit table | Report table | Finance dept. |
| Risk-reduction ranking | Sorted list | Emergency mgmt. |
| Interactive scenario explorer | Web app | Public / stakeholders |

The **spatial result** (which neighborhoods benefit most) is as important as the aggregate number. A UDT excels at making this geography visible.

---

## 🔑 Key Takeaways

1. Scenario analysis in a UDT = modifying one or more DAG nodes and propagating the change through the causal graph.
2. Scenarios are stored as **parallel tables** with `scenario_id`, enabling baseline vs. intervention comparisons via SQL joins.
3. **Spatial editing** (draw on map → trigger model) is the primary interface for urban planners to define intervention areas.
4. Results must be communicated as **maps** (which areas benefit) and **charts** (how much), not just aggregate numbers.

---

## 📎 References & Further Reading

- Weil, M., et al. (2023). Digital Twins for scenario-based urban planning. *Computers, Environment and Urban Systems*, 101.
- ArcGIS Urban Scenario Planning: https://www.esri.com/en-us/arcgis/products/arcgis-urban/
- Geopandas documentation (scenario spatial analysis): https://geopandas.org/

---

[← L13](lecture-13.md) · [Module 5 Index](index.md) · [Next: L15 – Participatory Digital Twins →](lecture-15.md)
