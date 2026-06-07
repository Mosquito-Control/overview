# Mosquito Control — Project Overview

Real-time drone airspace monitoring system. Detects drones via fixed cameras, triangulates their GPS position, tracks them over time, and alerts operators when drones enter restricted zones.

---

## How it works

```
Fixed cameras (RTSP)
  → System 1: YOLO detection → bearing vectors
  → System 2: multi-camera triangulation → GPS positions → PostgreSQL
  → System 4: greedy track association → persistent tracks → PostgreSQL
  → System 3: operator dashboard — map, zone alerts, track trails
```

All server components run on **Azure Container Apps**. Systems 2, 3, and 4 share a single **Azure PostgreSQL** instance as their integration boundary — no direct API coupling between systems.

---

## Repositories

| Repo | Role |
|---|---|
| [system1-detection-agent](https://github.com/Tion-ping/system1-detection-agent) | Per-camera YOLO inference · bearing vector POST to System 2 |
| [system2-positioning-engine](https://github.com/Tion-ping/system2-positioning-engine) | Ray triangulation · writes `positions` table |
| [system3-operator-dashboard](https://github.com/Tion-ping/system3-operator-dashboard) | Next.js map UI · zone alerts · track polylines |
| [system4-tracking-engine](https://github.com/Tion-ping/system4-tracking-engine) | Track association · writes `tracks` + `track_points` tables |

---

## Live URLs (Azure, westeurope)

| Service | URL |
|---|---|
| System 2 API | `https://system2-api.agreeablesea-31719cb5.westeurope.azurecontainerapps.io` |
| System 3 Dashboard | `https://system3-dashboard.agreeablesea-31719cb5.westeurope.azurecontainerapps.io` |
| System 4 API | `https://system4-tracking.agreeablesea-31719cb5.westeurope.azurecontainerapps.io` |

---

## Documentation in this repo

| File | Contents |
|---|---|
| [INTEGRATED_SYSTEM_STATE.md](./INTEGRATED_SYSTEM_STATE.md) | Full system state — architecture, endpoints, DB schema, config, risks |
| [RESEARCH.md](./RESEARCH.md) | ML model selection, detection/positioning approach, sprint plan |
| [DATASETS.md](./DATASETS.md) | Drone detection datasets ranked by relevance |
| [OS_COMPATIBILITY.md](./OS_COMPATIBILITY.md) | Simulator stack compatibility on Arch/CachyOS |
