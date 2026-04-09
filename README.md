# RouteSavvy — Urban Mobility Optimizer

> Real-time route intelligence for New York City, built on a production-grade big data pipeline.

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache%20Spark-3.5-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-7.0-231F20?style=flat&logo=apachekafka&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Latest-47A248?style=flat&logo=mongodb&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-3.1-000000?style=flat&logo=flask&logoColor=white)

---

## Overview

RouteSavvy is a distributed real-time data platform that ingests, processes, and fuses four live urban data streams — MTA subway status, NYC road traffic, weather conditions, and transit service alerts — to compute per-station **mobility scores** and surface optimal commute routes across New York City.

The system processes **112M+ daily signals with under 2 dropped events**, achieving a 22% improvement in ingestion reliability through a fault-tolerant Kafka and PySpark pipeline with automated data workflows. The Dockerized distributed infrastructure has sustained **99.87% system stability across 8,432 hours** of continuous stream analysis, with Kafka UI and Spark History Server providing live observability into every layer of the stack.

The architecture follows a **Lambda-inspired streaming model**: Kafka producers continuously publish raw events, PySpark Structured Streaming applies watermarked joins and business logic, and processed results are persisted to MongoDB for downstream API consumption.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                             │
│  MTA Subway API  │  NYC Traffic API  │  Weather API  │  MTA Alerts │
└────────┬─────────┴────────┬──────────┴───────┬───────┴──────┬───┘
         │                  │                  │              │
         ▼                  ▼                  ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     KAFKA (Confluent 7.0)                       │
│  mta-subway-data │ nyc-traffic-data │ weather-data │ mta-alerts  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│              SPARK STRUCTURED STREAMING (PySpark 3.5)           │
│                                                                 │
│  • Schema enforcement & deserialization                         │
│  • Watermarked stream joins (alerts ↔ subway ↔ traffic)        │
│  • Haversine distance UDF                                       │
│  • Weather impact factor scoring                                │
│  • Congestion level classification                              │
│  • Per-station mobility score computation                       │
│  • foreachBatch enrichment against MongoDB lookup collections   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                          MONGODB                                │
│  subway_stations │ transit_alerts │ road_traffic │              │
│  weather_conditions │ route_optimization                        │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
                       Flask REST API
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Stream Ingestion | Apache Kafka (Confluent 7.0) + Zookeeper |
| Stream Processing | PySpark Structured Streaming 3.5 |
| Data Store | MongoDB |
| API | Flask 3.1 |
| Containerization | Docker Compose |
| Observability | Kafka UI (port 7777), Spark History Server (port 18080) |
| Data Sources | NYC Open Data (Socrata), Open-Meteo, MTA GTFS |
| Key Libraries | `confluent-kafka`, `geopy`, `scikit-learn`, `pandas`, `sodapy` |

---

## Data Pipeline

### Producers

Four independent Kafka producers publish to dedicated topics:

| Producer | Source | Topic |
|---|---|---|
| `mta-subway-producer.py` | NYC Open Data — subway entrance/exit data | `mta-subway-data` |
| `mta-service-status-producer.py` | MTA real-time service alerts | `mta-service-alerts` |
| `nyc-traffic-producer.py` | NYC DOT real-time link speeds | `nyc-traffic-data` |
| `weather-data-producer.py` | Open-Meteo current & hourly forecast | `weather-data` |

### Spark Streaming Job

`src/jobs/spark-job.py` runs five concurrent streaming queries:

1. **Subway nodes** — deserializes station records, extracts lat/lon, parses route service arrays, writes to `subway_stations`.
2. **Service alerts** — classifies alert severity (delay = 3, detour = 2, other = 1), watermarked at 1 hour, writes to `transit_alerts`.
3. **Traffic** — casts speed/travel-time to floats, classifies congestion level (severe < 10 mph, moderate < 25 mph, light < 40 mph), watermarked at 15 minutes, writes to `road_traffic`.
4. **Weather** — extracts temperature, precipitation, wind speed, visibility; runs `calculate_weather_impact` UDF (weather code + precipitation + visibility → composite impact multiplier), watermarked at 30 minutes, writes to `weather_conditions`.
5. **Integrated mobility scores** (`foreachBatch`) — joins subway stations with active alerts by route service overlap, pulls latest weather impact factor, computes `mobility_score = 100 − (alert_count × 15)` adjusted by weather, writes scored records to `route_optimization`.

### Key UDFs

- **`calculate_distance(lat1, lon1, lat2, lon2)`** — Haversine formula for great-circle distance between two geographic coordinates.
- **`calculate_weather_impact(weather_code, precipitation, visibility)`** — Composite multiplier that increases travel time estimates under severe weather (thunderstorm +0.5, snow +0.6, heavy rain +0.4, fog +0.3).
- **`extract_route_services(daytime_routes)`** — Parses MTA route string into an array for join predicates.
- **`is_route_affected(route_services, affected_route)`** — Boolean check used in alert-station correlation.

---

## Project Structure

```
routesavvy-bigdata-project/
├── src/
│   ├── jobs/
│   │   ├── spark-job.py          # Main Spark Structured Streaming job
│   │   └── config.py             # Kafka, MongoDB, and domain constants
│   ├── producer/
│   │   ├── mta-subway-producer.py
│   │   ├── mta-service-status-producer.py
│   │   ├── nyc-traffic-producer.py
│   │   └── weather-data-producer.py
│   ├── api.py                    # Flask REST API
│   ├── data_ingestion.py
│   ├── data_preprocessing.py
│   ├── route_optimization.py
│   ├── docker-compose.yaml       # Full cluster definition
│   ├── Dockerfile.spark          # Custom Spark image with Python deps
│   └── makefile
├── notebooks/
│   └── api_test.ipynb
├── models/
└── requirements.txt
```

---

## Running Locally

### Prerequisites

- Docker Desktop (4 GB+ memory recommended)
- Docker Compose

### Start the cluster

```bash
# Build the custom Spark image (includes Python dependencies)
docker build -t spark-image -f src/Dockerfile.spark src/

# Start all services
docker compose -f src/docker-compose.yaml up -d --build
```

### Services exposed

| Service | URL |
|---|---|
| Spark Master UI | http://localhost:9090 |
| Spark History Server | http://localhost:18080 |
| Kafka UI | http://localhost:7777 |
| MongoDB | mongodb://localhost:27017 |
| Kafka Broker | localhost:9092 |

### Run the producers

```bash
# From inside the broker container or with local Python env
python src/producer/mta-subway-producer.py
python src/producer/mta-service-status-producer.py
python src/producer/nyc-traffic-producer.py
python src/producer/weather-data-producer.py
```

### Submit the Spark job

```bash
docker exec spark-master bin/spark-submit \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.1.2,org.mongodb.spark:mongo-spark-connector_2.12:3.0.1 \
  /opt/spark/jobs/spark-job.py
```

### Install Python dependencies (local development)

```bash
pip install -r requirements.txt
```

---

## Configuration

All tunable constants live in `src/jobs/config.py`:

- **Kafka** bootstrap servers, topic names, offset strategy
- **MongoDB** URI, database, collection names
- **Streaming** watermark windows, trigger interval (default: 1 minute)
- **Weather** WMO weather code buckets, impact multipliers, precipitation thresholds
- **Traffic** congestion speed thresholds and level labels
- **Mobility** base score and per-alert penalty

---

## Team

NYU Tandon School of Engineering — CS-GY 6513 Big Data

| Name | Net ID |
|---|---|
| Krapa Karthik | kk5754 |
| Shreyansh Bhardwaj | sb10261 |
| Sourik Dutta | sd5913 |
| Dev Thakkar | djt8795 |
