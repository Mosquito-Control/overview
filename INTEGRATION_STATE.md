# Integration State — Systems 1, 2, 3

**Date**: 2026-06-06  
**Status**: Systems 1→2→3 data path fully wired and verified

---

## System 1 → System 2: DONE ✓

System 1 (`Tion-ping/system1-detection-agent`) posts camera events to System 2's
`POST /events` endpoint. The Azure URL is hardcoded in System 1's `config.py`.
`cam_id` values (`cam_01`, `cam_02`) are consistent across both systems' `cameras.yaml`.
Verified end-to-end in a prior session: YOLO detect → POST → DB row within 1 second.

---

## System 2 → System 3: DONE ✓

System 3 (`Tion-ping/system3-operator-dashboard`) polls the `positions` table
via a Next.js server route at `/api/positions`. Poll cadence: 1.5s.

**DB credentials in `.env.local` (not committed):**
```
DATABASE_URL=postgresql://system3_reader:system3reader@drone-detection-pg.postgres.database.azure.com/dronedetection?sslmode=require
```

`system3_reader` role: SELECT on `positions` only. Password set by System 2 at startup
from the `READER_PASSWORD` env var (defaults to `system3reader`).

Verified: 7 positions in DB, `system3_reader` connects successfully.

---

## System 3 Alert Engine: DONE ✓

`src/lib/use-alert-engine.ts` — subscribes to the `drones` Zustand store slice.
On every position update (real or synthetic), runs `booleanPointInPolygon` against:
- All enabled **drawn zones** (operator-created polygons in the dashboard)
- All enabled **HK static no-fly zones** (from `public/data/hk-permanent-zones.geojson`)

Fires `pushAlert` on **zone entry only** (tracks active violations to avoid re-firing
every 1.5s while the drone stays inside).

Severity: `high` for military/airport/vip drawn zones and all static no-fly zones;
`medium` for event/country-park/custom drawn zones.

---

## Merged Data Streams

The dashboard runs two concurrent streams into the same `drones` store:

| Stream | Hook | IDs | Source |
|---|---|---|---|
| Real detections | `usePositions` | `cam_01+cam_02#<row_id>` | Azure PostgreSQL positions table |
| Synthetic background | `useDroneStream` | `SIM-001`, `SIM-002`, `SIM-003` | Circular orbits around HK center |

Both feed `useAlertEngine`. The ID prefix distinguishes real vs. simulated fixes
in the UI and alert messages.

---

## Camera Map Overlay

`public/data/cameras-seed.geojson` updated to include the two physical detection
cameras from System 2's `cameras.yaml`:

| cam_id | lat | lon | name |
|---|---|---|---|
| `cam_01` | 22.3193 | 114.1694 | Detection Camera 01 |
| `cam_02` | 22.3210 | 114.1694 | Detection Camera 02 |

---

## What Remains (System 4 / Future)

- System 4 (Unity simulation) is out of scope for this integration pass.
- The `cameras-seed.geojson` coordinates assume the Hong Kong reference frame from
  System 2's `cameras.yaml`. Update if physical camera locations change.
- `system3_reader` password (`system3reader`) is a default — rotate before any
  non-demo deployment.
