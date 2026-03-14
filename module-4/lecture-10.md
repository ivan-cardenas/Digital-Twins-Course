---
title: "L10 – Urban IoT Protocols: MQTT, NGSI-LD & FIWARE"
layout: default
---

# Lecture 10 — Urban IoT Protocols: MQTT, NGSI-LD & FIWARE

> 🕐 **Estimated duration:** 8–10 min  
> 📍 **Module 4 · Lecture 10 of 18**  
> 🎯 **Objective:** Connect live IoT sensor data to a UDT using MQTT and FIWARE's NGSI-LD standard

---

## 🎬 Lecture Overview

In this lecture you will learn:
- How IoT sensors communicate (MQTT protocol basics)
- What NGSI-LD is and why it matters for urban data interoperability
- FIWARE platform components relevant to UDTs
- A step-by-step example: ingesting an air quality sensor into a UDT

---

## 📖 Part 1 — MQTT and the IoT Messaging Model (~4 min)

### 10.1 The Challenge of Real-Time Urban Data

A city with 1,000 IoT sensors — air quality, traffic counters, noise meters, water pressure — generates data every few seconds. These sensors use low-power, lightweight protocols designed for constrained devices.

The most common: **MQTT** (Message Queuing Telemetry Transport).

### 10.2 MQTT: Publish-Subscribe Messaging

MQTT uses a **broker** model:
- **Publisher** (sensor) pushes readings to a broker on a **topic**
- **Subscriber** (UDT backend) listens to topics and receives messages

```
[Air Quality Sensor] → PUBLISH "city/sensors/aq/001" → [MQTT Broker]
                                                              ↓
                                          [UDT Backend SUBSCRIBES to topic]
                                                              ↓
                                                    Insert to PostGIS
```

**MQTT topic structure example:**
```
city/{district}/{sensor_type}/{sensor_id}
city/roombeek/air-quality/aq-042
city/centrum/noise/ns-017
city/enschede/water-pressure/wp-103
```

**Python MQTT subscriber:**
```python
import paho.mqtt.client as mqtt
import psycopg2, json

def on_message(client, userdata, msg):
    data = json.loads(msg.payload)
    cur.execute("""
        INSERT INTO sensor_readings (sensor_id, timestamp, pm25, no2, geom)
        VALUES (%s, NOW(), %s, %s, ST_SetSRID(ST_MakePoint(%s, %s), 4326))
    """, (data['id'], data['pm25'], data['no2'], data['lon'], data['lat']))
    conn.commit()

client = mqtt.Client()
client.on_message = on_message
client.connect("mqtt-broker", 1883)
client.subscribe("city/+/air-quality/#")
client.loop_forever()
```

### 10.3 NGSI-LD: Semantic Context Data Standard

**NGSI-LD** (Next Generation Service Interfaces - Linked Data) is an **ETSI/OGC standard** for representing and exchanging urban context data with semantic meaning.

An NGSI-LD entity looks like JSON-LD:

```json
{
  "@context": "https://smartdatamodels.org/context.jsonld",
  "id": "urn:ngsi-ld:AirQualityObserved:aq-042",
  "type": "AirQualityObserved",
  "location": {
    "type": "GeoProperty",
    "value": { "type": "Point", "coordinates": [6.895, 52.218] }
  },
  "PM2.5": { "type": "Property", "value": 18.4, "unitCode": "GQ" },
  "NO2": { "type": "Property", "value": 42.1, "unitCode": "GQ" },
  "observedAt": "2024-06-15T10:30:00Z"
}
```

Key advantages over plain JSON:
- **Semantic interoperability** — entities link to shared ontologies
- **GeoProperty** built in — location is a first-class attribute
- **Temporal** by default — `observedAt` is standard

---

## ✅ Mid-Lecture Quiz

> **Stop recording / pause here.**

**Question 1 — Single choice**

In an MQTT architecture, what component receives all sensor messages and routes them to subscribers?

- A) Publisher
- B) Topic
- C) **Broker** ✅
- D) Queue

**Question 2 — True or False**

NGSI-LD entities always include location as a built-in GeoProperty, making them naturally compatible with geospatial databases.

- A) **True** ✅
- B) False

**Question 3 — Short reasoning**

Why is NGSI-LD preferable to plain JSON for urban IoT data exchange across departments?

> ✅ **Expected answer:** NGSI-LD provides semantic interoperability through linked data contexts — different departments and systems can share data knowing it refers to the same concepts, reducing integration ambiguity.

---

## 📖 Part 2 — FIWARE Platform for Urban Digital Twins (~4 min)

### 10.4 FIWARE: The Urban Data Platform

**FIWARE** is an open-source platform developed under EU funding, designed for Smart City and IoT applications. Its core component is the **Orion Context Broker** — an NGSI-LD-compliant server that manages city-wide context data.

**FIWARE components relevant to UDTs:**

| Component | Role |
|-----------|------|
| **Orion-LD** (Context Broker) | Central hub for NGSI-LD entity management |
| **QuantumLeap** | Time-series storage (connects Orion → CrateDB/PostgreSQL) |
| **Cygnus** | Sink connector (Orion → PostGIS, MongoDB, Kafka) |
| **IoT Agent** | Translates MQTT/HTTP sensor data into NGSI-LD |
| **Grafana + STH-Comet** | Time-series visualization |

### 10.5 The Full IoT-to-UDT Pipeline

```
[Sensor] → MQTT →
[IoT Agent] → NGSI-LD →
[Orion Context Broker] → subscriptions →
[Cygnus] → PostGIS (persistent storage) +
[QuantumLeap] → CrateDB (time series) +
[TiTiler] → dynamic tile rendering ←
[Mapbox GL JS / React frontend]
```

### 10.6 Docker Compose Setup (Minimal)

```yaml
services:
  orion:
    image: fiware/orion-ld
    ports: ["1026:1026"]
    environment:
      - ORIONLD_MONGO_HOST=mongo

  iot-agent-json:
    image: fiware/iotagent-json
    environment:
      - IOTA_MQTT_HOST=mosquitto
      - IOTA_CB_HOST=orion

  mosquitto:
    image: eclipse-mosquitto
    ports: ["1883:1883"]

  cygnus:
    image: fiware/cygnus-ngsi
    environment:
      - CYGNUS_POSTGRESQL_HOST=postgis_db
```

### 10.7 Connecting to the UDT Spatial Database

Once Cygnus writes sensor readings to PostGIS, they are immediately queryable:

```sql
-- Get latest air quality readings per neighborhood
SELECT
    n.name,
    AVG(s.pm25) AS avg_pm25,
    MAX(s.timestamp) AS last_update
FROM sensor_readings s
JOIN neighborhoods n ON ST_Within(s.geom, n.geom)
WHERE s.timestamp > NOW() - INTERVAL '1 hour'
GROUP BY n.name
ORDER BY avg_pm25 DESC;
```

---

## 🔑 Key Takeaways

1. **MQTT** is the dominant protocol for IoT-to-server communication: lightweight, publish-subscribe, broker-based.
2. **NGSI-LD** standardizes urban context data with semantic meaning and built-in geospatial support.
3. **FIWARE** provides a complete open-source platform (Orion Context Broker + IoT Agents + Cygnus) for connecting sensor networks to a UDT.
4. The end-to-end pipeline: Sensor → MQTT → IoT Agent → Orion-LD → Cygnus → PostGIS → TiTiler/GeoServer → Web Map.

---

## 📎 References & Further Reading

- FIWARE NGSI-LD API: https://fiware-orion.readthedocs.io/
- ETSI NGSI-LD Standard: https://www.etsi.org/technologies/internet-of-things
- FIWARE Smart Data Models: https://smartdatamodels.org/
- Eclipse Mosquitto MQTT: https://mosquitto.org/

---

[← L9](../module-3/lecture-09.md) · [Module 4 Index](index.md) · [Next: L11 – Kafka & Real-Time PostGIS →](lecture-11.md)
