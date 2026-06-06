# System 2 — Positioning Engine: Current State

**Date**: 2026-06-06  
**Status**: Deployed and verified on Azure

---

## What Is Built

FastAPI service that receives camera detection events, runs multi-camera ray triangulation, and writes drone positions to PostgreSQL. Verified end-to-end: two test events → position row in DB within 1 second, GPS accurate to <1m.

---

## Live Endpoints

| | |
|---|---|
| API base URL | `https://system2-api.agreeablesea-31719cb5.westeurope.azurecontainerapps.io` |
| POST detections | `POST /events` |
| Health check | `GET /health` |

---

## Integration Contracts

### POST /events — payload from System 1

```json
{
  "cam_id": "cam_01",
  "timestamp": "2026-06-06T12:30:00.123Z",
  "detections": [
    { "bearing_vector": [0.0, 0.938, 0.346], "score": 0.91 }
  ]
}
```

**bearing_vector** is a 3D ENU (East-North-Up) unit vector. Conversion from azimuth α (degrees clockwise from north) and elevation φ (degrees above horizon):

```
E = cos(φ) * sin(α)
N = cos(φ) * cos(α)
U = sin(φ)
```

**Rules for System 1:**
- `cam_id` must match an entry in `cameras.yaml` (`cam_01` or `cam_02` for now)
- `timestamp` must be UTC, NTP-synced — clock drift >0.5s will break time-window matching
- Empty `detections: []` is valid (no drone seen this frame)

### positions table — read by System 3

```sql
SELECT id, timestamp, lat, lon, alt_m, cam_pair, score_i, score_j, inserted_at
FROM positions
ORDER BY inserted_at DESC;
```

| column | type | notes |
|---|---|---|
| id | SERIAL | |
| timestamp | TIMESTAMPTZ | UTC time of the source detections |
| lat | DOUBLE PRECISION | WGS84 |
| lon | DOUBLE PRECISION | WGS84 |
| alt_m | DOUBLE PRECISION | metres above sea level |
| cam_pair | TEXT | e.g. `cam_01+cam_02` |
| score_i / score_j | DOUBLE PRECISION | ML detection confidence per camera |
| inserted_at | TIMESTAMPTZ | server write time |

---

## Database

| | |
|---|---|
| Host | `drone-detection-pg.postgres.database.azure.com` |
| DB | `dronedetection` |
| Admin user | `droneadmin` |
| System 3 read-only user | `system3_reader` (SELECT on `positions` only) |
| Passwords | ask Filipp |

---

## Azure Resources (resource group: `drone-detection`)

| Resource | Name | Region |
|---|---|---|
| PostgreSQL Flexible Server | `drone-detection-pg` | northeurope |
| Container Registry | `dronedetectionacr` | westeurope |
| Container Apps environment | `drone-detection-env` | westeurope |
| Container App | `system2-api` | westeurope |

---

## Configuration (env vars on the Container App)

| Param | Value | Meaning |
|---|---|---|
| `MAX_DISTANCE_M` | 500 | max metres from each camera to accept a ray intersection |
| `TIME_WINDOW_S` | 1.0 | seconds back in cache to look per triangulation run |
| `CACHE_SIZE` | 300 | max events held in memory across all cameras |
| `LOOP_INTERVAL_S` | 0.5 | how often the triangulation loop fires |
| `DB_FLUSH_INTERVAL_S` | 5.0 | how often camera events are flushed to DB |

---

## Camera Configuration (cameras.yaml in repo)

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

To add cameras: add entries here, redeploy. No code changes needed.

---

## Repository

`https://github.com/Tion-ping/system2-positioning-engine`

```
system2/
├── config.py       # all env-var-backed settings
├── models.py       # Pydantic schemas
├── cache.py        # thread-safe ring buffer
├── triangulation.py # skew-line midpoint + ENU↔GPS math
├── db.py           # psycopg2 pool, table init, upserts
├── loop.py         # triangulation + flush background threads
├── api.py          # POST /events, GET /health
└── main.py         # FastAPI app + lifespan wiring
cameras.yaml        # static camera registry
Dockerfile
docker-compose.yml  # local dev (postgres in Docker)
```

### Local dev

```bash
docker-compose up
# API at http://localhost:8000
# Postgres at localhost:5432
```

---

## Known Issues / Risks

- **Clock sync**: System 1 camera timestamps must be NTP-synced UTC. Drift breaks time-window matching silently.
- **Coordinate frame**: bearing vectors must be ENU as defined above. Wrong frame = wrong positions with no error.
- **Schema stability**: System 3 reads `positions` directly. Any column changes need coordination with Bogdan.
- **`cameras.yaml` sync**: System 1 `cam_id` values must match entries in the YAML. Mismatch = events silently dropped.
