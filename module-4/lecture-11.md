---
title: "L11 – Stream Processing: Apache Kafka & Real-Time PostGIS"
layout: default
---

# Lecture 11 — Stream Processing: Apache Kafka & Real-Time PostGIS

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 4 · Lecture 11 of 18**  
> 🎯 **Objective:** Process high-throughput urban data streams and write them to a spatial database in near-real-time

---

## 🎬 Lecture Overview

In this lecture you will learn:
- Why Kafka is used for high-volume urban data streams
- Kafka topics and consumer groups in a UDT context
- Using Kafka Connect with PostGIS for spatial stream ingestion
- Aggregating spatial stream data with tumbling windows

---

## 📖 Part 1 — Why Apache Kafka for Urban Data? (~4 min)

### 11.1 MQTT vs. Kafka: Different Scales

MQTT (from L10) is excellent for sensor → server communication. But when a city has:
- 10,000 sensors × 1 reading/second = **10,000 messages/second**
- Multiple consuming systems (UDT, analytics, alerts, archive)
- Need for **replay** (reprocess historical data)

MQTT alone becomes a bottleneck. **Apache Kafka** handles all of this.

### 11.2 Kafka Core Concepts for UDTs

| Concept | Urban Analogy |
|---------|--------------|
| **Topic** | A data channel (e.g., `city.sensors.temperature`) |
| **Producer** | Sensor / FIWARE Cygnus writing to a topic |
| **Consumer** | UDT backend reading from a topic |
| **Consumer Group** | Multiple services reading independently (PostGIS + alerting + ML model) |
| **Partition** | Parallel processing lanes within a topic |
| **Offset** | Position in the message stream (enables replay) |

### 11.3 Kafka Topics for an Urban Digital Twin

```
city.sensors.air-quality        ← continuous AQ readings
city.sensors.traffic            ← vehicle counts per camera
city.sensors.water-pressure     ← pipe network pressure
city.events.incidents           ← reported events (fires, floods)
city.eo.lst-updates             ← new satellite raster available
city.cadastre.change-events     ← building permit approvals
```

### 11.4 Kafka Connect: Automatic Sink to PostGIS

**Kafka Connect** allows writing Kafka topic data directly to PostGIS without custom code:

```json
{
  "name": "postgis-sink-aq",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url": "jdbc:postgresql://postgis:5432/citydb",
    "topics": "city.sensors.air-quality",
    "auto.create": "true",
    "insert.mode": "insert",
    "table.name.format": "sensor_readings_aq"
  }
}
```

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

A UDT needs to feed incoming sensor data to three separate services simultaneously: (1) PostGIS for storage, (2) an alerting system, and (3) a machine learning model. What Kafka feature enables this?

- A) Topics
- B) Producers
- C) **Consumer Groups** ✅ *(each service is an independent consumer group, reading the same topic)*
- D) Partitions

**Question 2 — True or False**

Kafka enables replaying historical messages — meaning a new consumer can read data that was produced hours or days ago.

- A) **True** ✅ *(messages are retained based on retention policy — hours to forever)*
- B) False

---

## 📖 Part 2 — Spatial Stream Processing (~4 min)

### 11.5 Spatial Aggregation with Tumbling Windows

A common UDT need: aggregate sensor readings per neighborhood over a rolling time window.

Using **ksqlDB** (Kafka's stream SQL engine):

```sql
-- Create a stream from raw AQ topic
CREATE STREAM aq_stream (
  sensor_id VARCHAR,
  lat DOUBLE,
  lon DOUBLE,
  pm25 DOUBLE,
  timestamp BIGINT
) WITH (KAFKA_TOPIC='city.sensors.air-quality', VALUE_FORMAT='JSON');

-- Compute 5-minute average PM2.5 per sensor
CREATE TABLE aq_5min_avg AS
SELECT
  sensor_id,
  AVG(pm25) AS avg_pm25,
  WINDOWSTART AS window_start
FROM aq_stream
WINDOW TUMBLING (SIZE 5 MINUTES)
GROUP BY sensor_id
EMIT CHANGES;
```

### 11.6 Geofencing Alerts from the Stream

A critical UDT capability: detect when a sensor reading exceeds a threshold **within a specific geographic zone**:

```python
from kafka import KafkaConsumer
import json, psycopg2

consumer = KafkaConsumer('city.sensors.air-quality')

for msg in consumer:
    data = json.loads(msg.value)
    
    # Spatial query: which alert zone contains this sensor?
    cur.execute("""
        SELECT az.name, az.threshold_pm25
        FROM alert_zones az
        WHERE ST_Within(
            ST_SetSRID(ST_MakePoint(%s, %s), 4326),
            az.geom
        )
        AND %s > az.threshold_pm25
    """, (data['lon'], data['lat'], data['pm25']))
    
    for zone in cur.fetchall():
        trigger_alert(zone['name'], data)
```

### 11.7 Architecture Summary: Real-Time UDT Data Flow

```
[IoT Sensors] → MQTT → [FIWARE IoT Agent] → NGSI-LD
                                              ↓
                                    [Orion Context Broker]
                                              ↓
                                [Cygnus → Kafka Producer]
                                              ↓
                                    [Kafka Cluster]
                               ┌──────────┼──────────┐
                               ↓          ↓          ↓
                          [PostGIS]  [Alert Svc]  [ML Model]
                               ↓
                        [TiTiler/GeoServer]
                               ↓
                        [Map Dashboard]
```

---

## 🔑 Key Takeaways

1. **Kafka** handles high-throughput urban data streams that exceed MQTT's direct-to-database approach.
2. **Consumer Groups** allow multiple services (storage, alerting, ML) to independently consume the same data stream.
3. **ksqlDB** enables SQL-based stream processing — aggregations, filters, and joins on live spatial data.
4. **Geofencing** — checking if a stream event falls within a spatial zone — is a core real-time UDT capability implemented via PostGIS queries on incoming events.

---

[← L10](lecture-10.md) · [Module 4 Index](index.md) · [Next: L12 – Live Sensor Dashboard →](lecture-12.md)
