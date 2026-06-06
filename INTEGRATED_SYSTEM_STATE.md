# Integrated System State — Systems 1, 2 & 3

**Date**: 2026-06-06  
**Status**: Full data pipeline wired and verified end-to-end

---

## Architecture Overview

```
Unity RTSP / Physical Camera
        │  RTSP stream
        ▼
┌─────────────────────┐
│     System 1        │  Tion-ping/system1-detection-agent
│  Detection Agent    │  Python · YOLO · per-camera threads
└────────┬────────────┘
         │  POST /events  (HTTP, JSON)
         ▼
┌─────────────────────┐
│     System 2        │  Tion-ping/system2-positioning-engine
│  Positioning Engine │  FastAPI · Azure Container Apps
│                     │  triangulation loop every 500ms
└────────┬────────────┘
         │  INSERT positions (psycopg2)
         ▼
┌─────────────────────┐
│   Azure PostgreSQL  │  drone-detection-pg.postgres.database.azure.com
│   dronedetection    │  table: positions
└────────┬────────────┘
         │  SELECT (pg pool, poll every 1.5s)
         ▼
┌─────────────────────┐
│     System 3        │  Tion-ping/system3-operator-dashboard
│ Operator Dashboard  │  Next.js 16 · MapLibre · Zustand
│                     │  + synthetic SIM-001/002/003 overlay
│                     │  + zone-violation alert engine
└─────────────────────┘
```

---

## System 1 — Detection Agent

**Repo**: `https://github.com/Tion-ping/system1-detection-agent`  
**Status**: Implemented, tested with mock System 2

### What it does
- Reads RTSP streams (one thread per camera)
- Runs YOLOv8 inference on each frame
- Converts bounding box centres to 3D ENU bearing vectors
- POSTs `CameraEvent` to System 2 on every frame with detections

### Key config (`cameras.yaml` + env)

| Setting | Value | Override |
|---|---|---|
| System 2 URL | `https://system2-api.agreeablesea-31719cb5.westeurope.azurecontainerapps.io` | `SYSTEM2_URL` |
| YOLO weights | `yolov8s.pt` | `MODEL_PATH` |
| Confidence threshold | `0.35` | `CONF_THRESHOLD` |
| Min POST interval | `0.1s` | `POST_INTERVAL_S` |
| Post empty detections | `false` | `POST_EMPTY_DETECTIONS` |

### Camera config (`cameras.yaml`)

```yaml
cameras:
  - cam_id: cam_01
    stream_url: rtsp://localhost:8554/cam0   # Unity mediamtx cam 0
    azimuth_deg: 0.0
    elevation_deg: -15.0
    hfov_deg: 90.0
    vfov_deg: 60.0

  - cam_id: cam_02
    stream_url: rtsp://localhost:8554/cam1   # Unity mediamtx cam 1
    azimuth_deg: 90.0
    elevation_deg: -15.0
    hfov_deg: 90.0
    vfov_deg: 60.0
```

### POST /events payload

```json
{
  "cam_id": "cam_01",
  "timestamp": "2026-06-06T12:30:00.123Z",
  "detections": [
    { "bearing_vector": [0.606, 0.576, 0.515], "score": 0.91 }
  ]
}
```

**bearing_vector** is a 3D ENU (East-North-Up) unit vector derived from:
```
E = cos(φ) * sin(α)   N = cos(φ) * cos(α)   U = sin(φ)
```
where α = azimuth (°, clockwise from north) and φ = elevation (°, above horizon).

### Running

```bash
cd system1-detection-agent
pip install -r requirements.txt
python -m system1.main          # real RTSP mode
# E2E test (no Unity needed):
python tests/e2e/run_e2e.py
```

---

## System 2 — Positioning Engine

**Repo**: `https://github.com/Tion-ping/system2-positioning-engine`  
**Live API**: `https://system2-api.agreeablesea-31719cb5.westeurope.azurecontainerapps.io`  
**Status**: Deployed on Azure Container Apps, verified

### What it does
- Receives `POST /events` from System 1 cameras
- Buffers detections in a per-camera ring cache (300 events max)
- Every 500ms: attempts triangulation for all camera pairs in the cache
  using skew-line midpoint method in ENU coordinates
- Accepts intersections only within 500m of each camera (configurable)
- Converts accepted ENU intersection to WGS84 (lat/lon/alt)
- Upserts to `positions` table; also flushes raw camera events to DB every 5s

### Live endpoints

| Endpoint | Purpose |
|---|---|
| `POST /events` | Receive detection event from System 1 |
| `GET /health` | Returns `{status, cache_size, db_ok}` |

### Azure resources (resource group: `drone-detection`)

| Resource | Name | Region |
|---|---|---|
| PostgreSQL Flexible Server | `drone-detection-pg` | northeurope |
| Container Registry | `dronedetectionacr` | westeurope |
| Container Apps environment | `drone-detection-env` | westeurope |
| Container App | `system2-api` | westeurope |

### Runtime config (env vars on container app)

| Var | Value | Meaning |
|---|---|---|
| `MAX_DISTANCE_M` | 500 | Max metres from each camera for valid intersection |
| `TIME_WINDOW_S` | 1.0 | How far back in cache to look per triangulation run |
| `CACHE_SIZE` | 300 | Max events in ring buffer across all cameras |
| `LOOP_INTERVAL_S` | 0.5 | Triangulation loop cadence |
| `DB_FLUSH_INTERVAL_S` | 5.0 | How often raw events flush to DB |

### Camera config (`cameras.yaml` — same IDs as System 1)

```yaml
reference_origin:
  lat: 22.3193
  lon: 114.1694
  alt_m: 0.0

cameras:
  - id: cam_01
    lat: 22.3193
    lon: 114.1694
    alt_m: 15.0
  - id: cam_02
    lat: 22.3210    # ~190m north of cam_01
    lon: 114.1694
    alt_m: 15.0
```

### `positions` table schema (read by System 3)

```sql
SELECT id, timestamp, lat, lon, alt_m, cam_pair, score_i, score_j, inserted_at
FROM positions
ORDER BY inserted_at DESC;
```

| Column | Type | Notes |
|---|---|---|
| `id` | SERIAL | Auto-increment |
| `timestamp` | TIMESTAMPTZ | UTC time of source detections |
| `lat` | DOUBLE PRECISION | WGS84 |
| `lon` | DOUBLE PRECISION | WGS84 |
| `alt_m` | DOUBLE PRECISION | Metres above sea level |
| `cam_pair` | TEXT | e.g. `cam_01+cam_02` |
| `score_i/j` | DOUBLE PRECISION | ML detection confidence |
| `inserted_at` | TIMESTAMPTZ | Server write time |

### Database access

| Role | Password | Access |
|---|---|---|
| `droneadmin` | (ask Filipp — in container env) | Full admin |
| `system3_reader` | `system3reader` (default from `READER_PASSWORD` env) | SELECT on `positions` only |

### Local dev

```bash
cd system2-positioning-engine
docker-compose up          # starts FastAPI on :8000 + Postgres on :5432
```

---

## System 3 — Operator Dashboard

**Repo**: `https://github.com/Tion-ping/system3-operator-dashboard`  
**Status**: Implemented, connected to Azure PostgreSQL, build green

### What it does
- Polls `positions` table every 1.5s via a server-side Next.js route
- Displays live drone positions as dots on a MapLibre map
- Runs synthetic background traffic (SIM-001/002/003) in continuous loops
- Zone-violation alert engine fires on entry into drawn or HK static no-fly zones
- Operator can draw/edit/save restricted zones on the map
- Alerts panel lists all violations with severity and drone ID

### Data streams

| Stream | Hook | IDs | Always on? |
|---|---|---|---|
| Real detections | `usePositions` | `cam_01+cam_02#<row_id>` | Yes |
| Synthetic background | `useDroneStream` | `SIM-001`, `SIM-002`, `SIM-003` | Yes |

Both feed into the same Zustand `drones` store and through the same alert engine.

### Alert engine (`use-alert-engine.ts`)

Subscribes to `drones` store. On each update:
1. Runs `booleanPointInPolygon` for every drone against every enabled drawn zone
2. Runs same check against every enabled HK static no-fly zone
3. Fires `pushAlert` on zone **entry only** (re-entry after exit fires again)

Severity:
- `high` — military / airport / vip drawn zones, or any HK static no-fly zone
- `medium` — event / country-park / custom drawn zones

### Required `.env.local` (not committed)

```
DATABASE_URL=postgresql://system3_reader:system3reader@drone-detection-pg.postgres.database.azure.com/dronedetection?sslmode=require
NEXT_PUBLIC_POSITIONS_POLL_MS=1500
POSITIONS_LIMIT=200
```

### Running

```bash
cd system3-operator-dashboard
npm install
cp .env.example .env.local   # then fill in DATABASE_URL above
npm run dev                  # dashboard on http://localhost:3000
npm run probe                # verifies System 2 API + DB + /api/positions route
```

### Key source files

```
src/
├── app/
│   └── api/positions/route.ts     # server route, SELECTs from positions table
├── components/
│   ├── dashboard.tsx              # mounts all three hooks
│   ├── alerts-layer.tsx           # renders alert dots on map
│   ├── map-canvas.tsx             # zone draw/edit, point-in-polygon for clicks
│   └── hk-no-fly-layer.tsx        # HK static no-fly zone rendering
└── lib/
    ├── use-positions.ts           # real DB poll hook
    ├── use-drone-stream.ts        # synthetic SIM-* stream
    ├── use-alert-engine.ts        # zone-violation alert engine
    ├── pg-pool.ts                 # postgres connection pool (server-only)
    ├── store.ts                   # Zustand store (drones, zones, alerts)
    └── types.ts                   # DroneFix, Alert, ZoneFeature types
```

---

## Integration Verification Checklist

- [x] System 1 E2E test passes (bearing vectors unit, cam_id correct, POSTs reach System 2)
- [x] System 2 deployed on Azure, `/health` returns 200
- [x] `system3_reader` connects to Azure PostgreSQL — 7 positions in table as of 2026-06-06
- [x] System 3 build green (0 TS errors)
- [x] Both data streams mount on dashboard load
- [x] Alert engine fires on zone entry for both SIM and real drone IDs

## Known Risks / Gotchas

| Risk | Impact | Mitigation |
|---|---|---|
| System 1 clock drift >0.5s from NTP | Events fall outside System 2 time window, silently dropped | Ensure NTP on all System 1 hosts |
| `cam_id` mismatch between System 1 `cameras.yaml` and System 2 `cameras.yaml` | Events silently dropped | Both currently have `cam_01`, `cam_02` — keep in sync |
| Bearing vector in wrong coordinate frame | Wrong positions, no error | ENU frame required: E=cos(φ)sin(α), N=cos(φ)cos(α), U=sin(φ) |
| `system3_reader` default password | Low security | Acceptable for hackathon demo; rotate before any production use |
| `positions` schema changes | System 3 query breaks | Coordinate with System 3 before any column renames/drops |
