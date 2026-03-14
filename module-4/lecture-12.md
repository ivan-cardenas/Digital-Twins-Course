---
title: "L12 – Building a Live Sensor Dashboard"
layout: default
---

# Lecture 12 — Building a Live Sensor Dashboard

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 4 · Lecture 12 of 18**  
> 🎯 **Objective:** Build a React-based UDT dashboard that displays live sensor data on a map with charts

---

## 🎬 Lecture Overview

In this lecture you will learn:
- Dashboard architecture for a live UDT interface
- Connecting a React frontend to PostGIS via a Django/FastAPI backend
- Displaying live sensor data on a Mapbox GL JS map
- Updating charts in real-time using WebSockets or polling

---

## 📖 Part 1 — Dashboard Architecture (~4 min)

### 12.1 The Full-Stack Architecture

```
[PostGIS] ← writes ← [Kafka Consumer]
    ↓
[GeoDjango / FastAPI REST API]
    ↓ JSON (GeoJSON + time series)
[React Frontend]
    ├── Mapbox GL JS map (spatial view)
    ├── Chart.js / Recharts (time series)
    └── KPI cards (aggregated metrics)
```

### 12.2 GeoDjango API: Serving Sensor Data

Using Django REST Framework + GeoDjango:

```python
# models.py
from django.contrib.gis.db import models

class SensorReading(models.Model):
    sensor_id = models.CharField(max_length=50)
    timestamp = models.DateTimeField(auto_now_add=True)
    pm25 = models.FloatField(null=True)
    no2 = models.FloatField(null=True)
    location = models.PointField(srid=4326)

# views.py
from django.contrib.gis.geos import GEOSGeometry
from rest_framework_gis.serializers import GeoFeatureModelSerializer

class SensorReadingSerializer(GeoFeatureModelSerializer):
    class Meta:
        model = SensorReading
        geo_field = 'location'
        fields = ['sensor_id', 'timestamp', 'pm25', 'no2']
```

### 12.3 React Map Component with Live Data

```jsx
import { useState, useEffect } from 'react';
import Map, { Source, Layer } from 'react-map-gl';

export function SensorMap() {
  const [sensors, setSensors] = useState({ type: 'FeatureCollection', features: [] });

  useEffect(() => {
    const fetchSensors = async () => {
      const res = await fetch('/api/sensors/latest/?format=json');
      const data = await res.json();
      setSensors(data);
    };
    fetchSensors();
    const interval = setInterval(fetchSensors, 30000); // refresh every 30s
    return () => clearInterval(interval);
  }, []);

  return (
    <Map initialViewState={{ longitude: 6.9, latitude: 52.22, zoom: 12 }}>
      <Source id="sensors" type="geojson" data={sensors}>
        <Layer
          id="sensor-circles"
          type="circle"
          paint={{
            'circle-radius': 10,
            'circle-color': [
              'interpolate', ['linear'], ['get', 'pm25'],
              0, '#00ff00', 25, '#ffff00', 50, '#ff0000'
            ]
          }}
        />
      </Source>
    </Map>
  );
}
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

In a UDT dashboard, which component is responsible for translating database queries into HTTP API responses that the React frontend can consume?

- A) PostGIS
- B) Kafka
- C) **GeoDjango / FastAPI backend** ✅
- D) TiTiler

**Question 2 — Best approach**

For displaying a live PM2.5 reading that updates every 30 seconds, which update strategy is most appropriate for a simple dashboard?

- A) WebSockets (persistent connection)
- B) **Polling with setInterval** ✅ *(appropriate for 30s intervals; WebSockets for sub-second needs)*
- C) Server-Sent Events
- D) GraphQL subscriptions

---

## 📖 Part 2 — Time-Series Charts and KPI Cards (~4 min)

### 12.4 Time-Series Chart with Recharts

```jsx
import { LineChart, Line, XAxis, YAxis, Tooltip, Legend } from 'recharts';

function AQTimeSeries({ sensorId }) {
  const [data, setData] = useState([]);

  useEffect(() => {
    fetch(`/api/sensors/${sensorId}/history/?hours=24`)
      .then(r => r.json())
      .then(setData);
  }, [sensorId]);

  return (
    <LineChart width={600} height={300} data={data}>
      <XAxis dataKey="timestamp" />
      <YAxis />
      <Tooltip />
      <Legend />
      <Line type="monotone" dataKey="pm25" stroke="#ff4444" name="PM2.5 (μg/m³)" />
      <Line type="monotone" dataKey="no2"  stroke="#4444ff" name="NO₂ (μg/m³)" />
    </LineChart>
  );
}
```

### 12.5 KPI Cards from Spatial Aggregation

```python
# Django API view: KPIs per neighborhood
@api_view(['GET'])
def neighborhood_kpis(request, neighborhood_id):
    nb = Neighborhood.objects.get(id=neighborhood_id)
    readings = SensorReading.objects.filter(
        location__within=nb.geom,
        timestamp__gte=timezone.now() - timedelta(hours=1)
    )
    return Response({
        'avg_pm25': readings.aggregate(Avg('pm25'))['pm25__avg'],
        'max_pm25': readings.aggregate(Max('pm25'))['pm25__max'],
        'sensor_count': readings.values('sensor_id').distinct().count(),
        'last_updated': readings.latest('timestamp').timestamp
    })
```

### 12.6 Dashboard Design Principles for UDTs

A good UDT dashboard follows these principles:

| Principle | Implementation |
|-----------|---------------|
| **Context first** | Map always visible; charts are linked to selected spatial feature |
| **Multi-temporal** | Show current state AND historical trends |
| **Sector-aware** | Tabs or panels for water / heat / mobility |
| **Alert visibility** | Threshold breaches shown prominently on map |
| **Accessible** | Color-blind safe palettes; accessible labels |

---

## 🔑 Key Takeaways

1. A live UDT dashboard has three tiers: **PostGIS** (storage) → **GeoDjango/FastAPI** (API) → **React** (UI).
2. **Polling** (setInterval) is appropriate for 30-second sensor updates; **WebSockets** are needed for sub-5-second real-time.
3. **react-map-gl + Mapbox GL JS** handles the spatial map; **Recharts** handles time-series; **KPI cards** come from spatial aggregation queries.
4. Good UDT dashboard design always connects the **map** to the **chart** — clicking a sensor on the map populates the time-series panel.

---

[← L11](lecture-11.md) · [Module 4 Index](index.md) · [Next: Module 5 →](../module-5/index.md)
