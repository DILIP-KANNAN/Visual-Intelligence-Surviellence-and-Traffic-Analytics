# 🚦 Visual Intelligence, Surveillance and Traffic Analytics (VISTA / NeuralGrid)
## System Architecture & Feature Guide

> For comprehensive formal specifications, sequence flows, entity-relationship diagrams, and mathematical formulations, refer to [TECHNICAL_DESIGN.md](file:///TECHNICAL_DESIGN.md).

---

## 1. High-Level Architecture Diagram

```mermaid
flowchart TD
    subgraph Ingestion["Perception & Ingestion Layer"]
        CCTV["CCTV Edge AI (YOLO + PaddleOCR)"] -->|JSON Event| RedisStream["Redis Stream: anpr_events"]
        DemoStreamer["Live Scenario Streamer (demo_streamer.py)"] -->|Simulated ANPR Flow| RedisStream
        RedisStream --> Consumer["Redis Consumer (redis_consumer.py)"]
    end

    subgraph Storage["Storage & Spatiotemporal Engine"]
        Consumer --> PG_Events[("PostgreSQL + PostGIS\nevents / vehicles")]
        PG_Events -->|PostgreSQL Trigger| TrajectoryEngine["Trajectory Engine (trajectory_engine.py)"]
        TrajectoryEngine --> PG_Traj[("trajectories")]
        PG_Traj -->|pg_notify| TrajPublisher["Trajectory Publisher (trajectory_event_publisher.py)"]
    end

    subgraph Analytics["Continuous Analytical Engines"]
        TrajPublisher --> StreamTraj["Redis Stream: trajectory_events"]
        StreamTraj --> RouteAnomaly["Route Anomaly Worker (2σ Outliers)"]
        StreamTraj --> CongestionEngine["Congestion & Delay Engine (15-min Baselines)"]
        CongestionEngine --> Propagation["Delay Propagation Engine (Downstream Cascades)"]
        PG_Events --> CameraHealth["Camera Health Sentinel (Heartbeat Audits)"]
        PG_Events --> Convoy["Collective Movement Engine (Convoy Detection)"]
        RouteAnomaly & CameraHealth & Convoy --> PG_Alerts[("alerts")]
        PG_Alerts --> AuthorityEngine["Authority Assignment Engine (Police Divisions)"]
    end

    subgraph Realtime["Real-Time Gateway & UI"]
        TrajPublisher -->|trajectory_events / alert_events| FastAPIServer["FastAPI Gateway (/ws/traffic)"]
        AuthorityEngine --> FastAPIServer
        FastAPIServer --> Frontend["React + MapLibre Command Center\n(Live Map, Alerts, Vehicle Dossier, Analytics)"]
    end
```

---

## 2. How Does Data Enter Without Physical CCTV Cameras?

In a live smart city deployment, physical CCTV cameras run Edge AI (YOLO detection + PaddleOCR) to read license plates and push events into the ingestion pipeline.

In development, testing, and evaluation, we simulate this using the **Live Scenario Simulation Streamer** (`demo_streamer.py`) or the standalone producer client (`redis_producer.py`). 

### The Complete Event Pipeline:
```
[ LIVE CCTV / VIDEO STREAM ] (Production)
              OR                                ──> Emits JSON: { camera_id, license_plate, timestamp, confidence }
[ DEMO STREAMER (demo_streamer.py) ] (Dev/Demo)
              │
              ▼
   [ REDIS STREAM: anpr_events ]
              │
              ▼ (Batch Consumer)
      [ redis_consumer.py ]
              │
              ├──────────────────────────────────┐
              ▼                                  ▼
      [ PostgreSQL: events ]          [ PostgreSQL: trajectories ]
      Raw detections stored           Auto-links: CAM01 -> CAM02
                                                 │
                                                 ▼ (PostgreSQL Trigger)
                                      [ NOTIFY trajectory_created ]
                                                 │
                                                 ▼
                                   [ trajectory_event_publisher.py ]
                                                 │
                                                 ▼
                                     [ REDIS: trajectory_events ]
                                                 │
                                                 ▼
                                        [ app/realtime.py ]
                                                 │
                                                 ▼
                                    [ WEBSOCKET: /ws/traffic ]
                                                 │
                                                 ▼
                                     [ FRONTEND DASHBOARD ]
```

When you run `python demo_streamer.py` (or `python start_system.py --demo`), it continuously generates vehicle sightings across cameras (e.g. `CAM01`, `CAM02`, `CAM03` ... `CAM10`) along real Chennai highway routes. Once real YOLO/PaddleOCR camera feeds are connected, they push the identical JSON format to the Redis `anpr_events` stream—zero code changes required in the rest of the backend or frontend!

---

## 3. Detailed Feature Breakdown & Frontend Location

---

### Feature 1: Live Interactive GIS Map & Camera Network
* **What It Does**:
  Provides a real-time visual command center map displaying all CCTV camera nodes across Chennai, active road corridors, and live health statuses.
* **Backend Processing**:
  - `GET /cameras`: Fetches GPS coordinates and names for all cameras.
  - `GET /cameras/health`: Calculates minutes since last sighting for each camera and determines health status:
    - **HEALTHY** (< 15 min since last vehicle)
    - **WARNING** (15 – 60 min)
    - **OFFLINE** (> 60 min)
  - `GET /camera-routes`: Queries PostGIS `MultiLineString` geometries generated from OpenStreetMap road networks.
* **Where It Is Displayed on Frontend**:
  - **Tab**: `Live Map`
  - **Map Canvas**: High-contrast, dark-mode MapLibre GL map centered on Chennai.
  - **Camera Pins**: Colored circular markers on exact GPS coordinates:
    - 🟢 Green = Healthy
    - 🟡 Yellow/Orange = Warning
    - 🔴 Red = Offline
    - Clicking any pin displays a popup with camera ID, name, configured status, and health.
  - **Corridor Lines**: Glowing blue lines connecting camera nodes along major roads (Anna Salai, Guindy, OMR, etc.).
  - **Top KPI Cards**: 3 cards showing **Total Cameras**, **Healthy Cameras**, and **Offline Cameras**.
  - **Side Network Card**: Lists healthy, warning, and offline camera breakdowns.

---

### Feature 2: Real-Time Vehicle Trajectory Tracking
* **What It Does**:
  As vehicles pass consecutive CCTV cameras, the system links sequential sightings into multi-camera journeys (trajectories) with measured travel times and distances.
* **Backend Processing**:
  - `redis_consumer.py` matches vehicle sightings by license plate within realistic time windows ($t_{start} \to t_{end}$).
  - Inserts completed hops into `trajectories` table.
  - PostgreSQL trigger `trigger_trajectory_created` fires `pg_notify('trajectory_created')`.
  - `trajectory_event_publisher.py` pushes to Redis `trajectory_events`.
  - `app/realtime.py` pushes JSON over `/ws/traffic` WebSocket to browser clients.
* **Where It Is Displayed on Frontend**:
  - **Tab**: `Live Map` (Floating Card) & `Vehicles`
  - **Live Movement Card** (Bottom-right of `Live Map`):
    - Displays: `Latest movement`
    - Large Vehicle ID: e.g. `TN09BV1907`
    - Route hop: `CAM001 → CAM002`
    - Subtitle: `Listening for ANPR events…`
  - **`Vehicles` Tab**:
    - Master table of all vehicles seen across Chennai.
    - Columns: Plate Number, Vehicle Type, Route, Speed, Last Seen timestamp, and Status badge (`Authorized`, `Flagged`, `Suspicious`).

---

### Feature 3: Route Travel-Time Anomaly Detection
* **What It Does**:
  Detects vehicles taking abnormal detours, experiencing unexpected delays/breakdowns, speeding, or skipping intermediate mandatory camera checkpoints.
* **Backend Processing**:
  - `route_anomaly_worker.py`: Compares each vehicle's `travel_time_seconds` against baseline statistics (`route_travel_stats`).
  - Statistical threshold: Flags if travel time deviates beyond $Mean \pm 2\sigma$.
  - Inserts alert into PostgreSQL `alerts` table (`ROUTE_ANOMALY`).
  - PostgreSQL trigger `trigger_alert_created` fires and broadcasts `ALERT_CREATED` over WebSockets.
* **Where It Is Displayed on Frontend**:
  - **Sidebar Navigation Badge**: The `Alerts` tab button displays a red badge with the real-time active count (e.g. `Alerts [14]`).
  - **`Alerts` Tab**:
    - Color-coded incident cards (Red = Critical, Orange = High, Yellow = Medium).
    - Shows: Alert Type (`ROUTE_ANOMALY`), Severity, Cause message, Vehicle ID, Camera ID, and timestamp.
  - **`Live Map` Tab**: An alert ticker badge appears directly inside the live movement box:
    - `🚨 LIVE ALERT: ROUTE_ANOMALY (CAM002)`

---

### Feature 4: Camera Health & Heartbeat Monitor
* **What It Does**:
  Audits the CCTV network every 60 seconds. If a camera stops detecting vehicles (due to hardware failure, network disconnection, or camera occlusion), it alerts operators immediately.
* **Backend Processing**:
  - `camera_health_worker.py`: Runs every 60s, querying `MAX(observed_at)` for each camera.
  - If delta > 60 min, changes status to `OFFLINE` and creates a `CAMERA_HEALTH` alert.
  - Publishes `CAMERA_HEALTH_UPDATED` event to Redis stream `camera_health_events`.
  - Broadcasts via WebSocket to frontend.
* **Where It Is Displayed on Frontend**:
  - **`Live Map` Tab**:
    - The camera pin on the map switches from Green to Orange or Red dynamically without page reload.
    - Top KPI cards update ("Healthy Cameras" count decreases, "Offline Cameras" increases).
  - **`Alerts` Tab**:
    - Creates an incident card: `CAMERA_HEALTH | WARNING: No events received from CAM003 for 64 minutes`.

---

### Feature 5: Collective Movement & Convoy Detection
* **What It Does**:
  Detects unusual surges or coordinated convoys (groups of $\ge 3$ vehicles traveling together within a tight time window across consecutive cameras).
* **Backend Processing**:
  - `collective_movement_engine.py`: Scans 15-minute windows for multi-vehicle clusters.
  - Flags clusters representing $\ge 70\%$ of current traffic but $< 30\%$ historically.
  - Inserts `COLLECTIVE_MOVEMENT` alert into `alerts` table.
* **Where It Is Displayed on Frontend**:
  - **`Alerts` Tab**:
    - Incident card displaying `COLLECTIVE_MOVEMENT`, specifying convoy vehicle count, corridor, and timestamp.

---

### Feature 6: Historical Congestion & Delay Trends
* **What It Does**:
  Aggregates traffic patterns across 15-minute time windows to calculate average delays, travel time index (TTI), and congestion severity.
* **Backend Processing**:
  - `congestion_engine.py` / `analytics_worker.py`: Computes historical baseline vs. current window speeds.
  - Computes `average_delay_seconds` and `congestion_level` (`LOW`, `MODERATE`, `HIGH`, `CRITICAL`).
  - Stored in `historical_congestion` table.
  - `GET /analytics/congestion/history`: Queried by frontend.
* **Where It Is Displayed on Frontend**:
  - **Tab**: `Analytics`
  - **Traffic Flow Chart** (Top-Left): Interactive SVG polyline chart showing total vehicle count per 15-minute window.
  - **Congestion Trend Chart** (Bottom-Right): Trend graph showing average delay percentage over time to identify bottleneck hours.

---

### Feature 7: Monitored Zone Density Analytics
* **What It Does**:
  Tracks vehicle volumes across city sectors (e.g., Central Business District, Anna Salai Zone, Guindy Industrial Corridor) using PostGIS spatial boundaries.
* **Backend Processing**:
  - Spatial containment query: `ST_Contains(zone.boundary, camera.geom)`.
  - `GET /analytics/traffic/zones`: Groups event counts by zone ID.
* **Where It Is Displayed on Frontend**:
  - **Tab**: `Analytics`
  - **Vehicle Density Chart** (Bottom-Left): Compares vehicle distribution across zones (`Z1`, `Z2`, etc.) so authorities know which sectors are overburdened.

---

### Feature 8: Origin-Destination (OD) Commuter Matrix
* **What It Does**:
  Tracks complete commuter trips from city entry cameras to exit cameras to analyze macro traffic flows.
* **Backend Processing**:
  - `od_engine.py`: Reconstructs trips from continuous vehicle trajectories.
  - `GET /analytics/od-matrix`: Aggregates trip counts per `(origin_camera, destination_camera)` pair.
* **Where It Is Displayed on Frontend**:
  - **Tab**: `Analytics`
  - Feeds traffic volume matrix cards showing primary transit corridors across Chennai.

---

### Feature 9: Traffic Authority Alert Assignment
* **What It Does**:
  Automatically routes incidents to the relevant local traffic police division based on GIS zone boundaries.
* **Backend Processing**:
  - Joins `alerts` with `zones.authority_name` (e.g. "Anna Salai Traffic Wing", "Guindy Division").
  - `GET /analytics/alerts/authorities`: Returns alert load per police department.
* **Where It Is Displayed on Frontend**:
  - **`Alerts` Tab**: Inside each alert card, the location tag displays the assigned division (e.g., `⌖ Guindy Division`).

---

## 3. Quick Reference Matrix: Feature to Frontend Tab

| Feature | Backend Component | Primary Frontend Tab | UI Element |
| :--- | :--- | :--- | :--- |
| **Camera Network** | `cameras` table, `/cameras` API | `Live Map` | MapLibre Map Pins (Green/Yellow/Red) |
| **Camera Health** | `camera_health_worker.py`, `/cameras/health` | `Live Map` | Top KPI Cards (Total/Healthy/Offline) |
| **Corridor Paths** | `camera_routes` table, PostGIS GeoJSON | `Live Map` | Glowing Blue Route Overlays |
| **Live Movements** | `trajectories`, Redis Stream, `/ws/traffic` | `Live Map` | Floating "Latest Movement" Glass Card |
| **Vehicle Directory** | `events`, `trajectories`, `/vehicles` | `Vehicles` | Filterable Searchable ANPR Table |
| **Route Anomalies** | `route_anomaly_worker.py`, `alerts` | `Alerts` | Red Badge on Sidebar + Incident Cards |
| **Convoy Detection**| `collective_movement_engine.py` | `Alerts` | Incident Cards (`COLLECTIVE_MOVEMENT`) |
| **Traffic Flow** | `congestion_engine.py`, `analytics_worker.py` | `Analytics` | Polyline Flow Chart (Vehicles / Window) |
| **Delay Trends** | `historical_congestion` table | `Analytics` | Delay Percentage Trend Chart |
| **Zone Density** | PostGIS Polygon Containment, `zones` | `Analytics` | Zone Density Bar/Spark Chart |
| **OD Matrix** | `od_engine.py`, `trips` table | `Analytics` | OD Flow Aggregations |
