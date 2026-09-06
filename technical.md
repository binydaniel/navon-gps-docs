# Design and Architecture

A self-hosted fleet management & telemetry platform. It ingests GPS data from physical tracking devices (Teltonika FMB series, Queclink, GT06), processes telemetry in real-time, and presents fleet intelligence through a web dashboard, an Android mobile app, and a standalone GPS simulator for testing.

***

## Documentation

| Document                          | Description                                                       |
| --------------------------------- | ----------------------------------------------------------------- |
| [README](technical.md)            | Getting started, architecture, and quick references               |
| [SRS](technical/SRS.md)           | Full software requirements specification (features, APIs, schema) |
| [WORKFLOW](technical/workflow.md) | Onboarding and operational workflow for all roles                 |

***

## Architecture

```
Teltonika / Queclink / GT06 Device ──(TCP)──► DeviceTcpListener (BackgroundService)
                                                        │
                                               IDeviceProtocol Parser
                                                        │
                                          ProcessGpsTelemetry (MediatR)
                                                        │
                               ┌────────────┬───────────┴──────────┐
                               │            │                      │
                        Postgres+PostGIS  GeofenceCheckService  NotificationEvaluator
                        (Dapper, geography)      │                      │
                               │            LiveTrackingHub (SignalR)   │
                           REST API (/api)             │                │
                                                    Web / Mobile clients
```

Everything runs inside one ASP.NET Core process (`MyGpsTracker.Api`) so device ingestion never blocks HTTP requests and vice versa. TCP listeners are `BackgroundService`s running alongside Kestrel.

### Auth & Tenancy

* **ASP.NET Core Identity** for account management (register, login, password change).
* **JWT Bearer** tokens (stateless), validated on every REST and SignalR request.
* **Role-based access**: `SystemAdmin` (global), `Admin`, `User`, `Driver` (company-scoped).
* **Multi-tenant** by company: devices, vehicles, users, and geofences are isolated per company.

***

## Quick Start

### Prerequisites

* Docker + Docker Compose
* Or: .NET 9 SDK + PostgreSQL/PostGIS locally

### Run with Docker

```bash
cd GPSTracker
docker-compose up --build
```

| Service            | Host URL                                       |
| ------------------ | ---------------------------------------------- |
| Web dashboard      | http://localhost:8082                          |
| REST API + Swagger | http://localhost:8081/swagger                  |
| SignalR Hub        | ws://localhost:8081/hubs/tracking              |
| Seq (logs)         | http://localhost:8083                          |
| TCP ingestion      | 5027 (Teltonika), 5011 (Queclink), 5023 (GT06) |
| PostgreSQL         | localhost:5433 (db: `gpstracker`)              |

Environment variables:

| Variable            | Description                              |
| ------------------- | ---------------------------------------- |
| `POSTGRES_PASSWORD` | Database password (required)             |
| `JWT_KEY`           | JWT signing key, min 32 chars (required) |

No separate migration step — `db/init.sql` runs automatically the first time the `db` container starts (creates the `postgis` extension and all tables). The API bootstraps the base admin + default company on first start.

### Run without Docker

```bash
# 1. Start Postgres+PostGIS, then apply schema:
psql "postgresql://gpstracker:gpstracker@localhost:5432/gpstracker" -f db/init.sql

# 2. Run the API (set ConnectionStrings__DefaultConnection and Jwt__Key)
cd api/MyGpsTracker.Api
dotnet run
```

> **Note on .NET version:** the API targets `net9.0`. Bump `<TargetFramework>` to `net10.0` once you have the .NET 10 SDK locally — nothing else changes.

***

## Projects

| Path                                                     | Description                                                            |
| -------------------------------------------------------- | ---------------------------------------------------------------------- |
| [`api/MyGpsTracker.Api`](technical/api/MyGpsTracker.Api) | ASP.NET Core 9 backend — CQRS/MediatR + Dapper, TCP ingestion, SignalR |
| [`db`](technical/db/)                                    | PostGIS schema (Docker init)                                           |
| [`web`](technical/web/)                                  | React 18 + TypeScript + Vite + Tailwind dashboard (maps via Leaflet)   |
| [`web/android`](technical/web/android/)                  | Capacitor hybrid Android app (biometric auth, secure storage)          |
| [`gps-simulator`](technical/gps-simulator/)              | Standalone Android fleet simulator (Teltonika Codec 8 over TCP)        |

### Key Backend Features

| Feature                 | Location                                                         |
| ----------------------- | ---------------------------------------------------------------- |
| TCP device listener     | `api/MyGpsTracker.Api/Ingestion/Sockets/DeviceTcpListener.cs`    |
| Teltonika Codec parser  | `api/MyGpsTracker.Api/Ingestion/Teltonika/`                      |
| Telemetry pipeline      | `api/MyGpsTracker.Api/Features/Telemetry/ProcessGpsTelemetry.cs` |
| Devices / vehicles      | `api/MyGpsTracker.Api/Features/Devices/`                         |
| Geofences + violations  | `api/MyGpsTracker.Api/Features/Geofences/`                       |
| Notifications engine    | `api/MyGpsTracker.Api/Features/Notifications/`                   |
| Live tracking (SignalR) | `api/MyGpsTracker.Api/Features/Tracking/LiveTrackingHub.cs`      |
| Auth / roles / JWT      | `api/MyGpsTracker.Api/Features/Auth/`                            |
| Admin / companies       | `api/MyGpsTracker.Api/Features/Admin/`                           |
| DB bootstrap            | `api/MyGpsTracker.Api/Infrastructure/DatabaseBootstrap.cs`       |

### Database Bootstrap

The schema and seed logic lives in `Infrastructure/DatabaseBootstrap.cs` (replaces the old inline `Program.cs` SQL). On startup it:

1. Applies pending Identity migrations.
2. Seeds `SystemAdmin` + `Default` company.
3. Applies the devices/telemetry schema handled by `db/init.sql`.

***

## Features

### Device Management (F-02)

* Auto-registration of physical devices on first TCP connection
* Device list / detail / history / trips / events / power log
* Device search, edit display name, offline threshold
* Multi-company device assignment (`company_devices`)
* Vehicle assignment (1:1 via `vehicles.device_id`) and SIM phone number

### Live Vehicle Tracking (F-03)

* Leaflet map with live markers via SignalR (`LiveTrackingHub`)
* Marker colors: green = external power, yellow = battery, gray = offline
* Ignition ON/OFF badges, device sidebar, offline detection
* Telemetry detail: DIN 1-4, DOUT 1-3, accelerometer XYZ, GSM signal, odometer, sleep mode, movement

### Geofencing (F-04)

* Circle and polygon geofences
* Enter/exit detection in real-time (`GeofenceCheckService` + state cache)
* Queue-server-side per-geofence speed limit + violations tracking
* Active violations list and recent events

### Remote Device Configuration — Codec 12 (F-05)

* Send arbitrary commands (`setparam`, `getinfo`, `getver`, ...) to online devices
* Quick command buttons, CPU reset (with confirm), engine immobilizer (`setdigout 1/0`)
* In-memory connection registry maps IMEI → live TCP stream

### Notifications Engine (F-06)

* Threshold-based alerts: speeding, hard\_brake, hard\_curve, low\_battery, offline, geofence\_speeding
* 2-minute cooldown per device+type; real-time push via SignalR
* Persistent notifications with severity / read status; unread badge

### Eco Scoring & Driving Events (F-07)

* Harsh acceleration, hard braking, hard cornering, idling, movement-while-off, signal jamming, low GSM signal

### Power Monitoring (F-09) & Digital I/O (F-10)

* External/battery voltage → power source computation
* Power log endpoint (24h voltage history)
* DIN 1-4 / DOUT 1-3 live via SignalR

### GPS Simulator App (F-11)

Standalone Android app (`gps-simulator/`) that simulates one or many Teltonika devices:

| Feature                        | Description                                                            |
| ------------------------------ | ---------------------------------------------------------------------- |
| Fleet simulation               | Runs many concurrent devices (Quick Fleet)                             |
| Smooth route simulation        | Devices drive along a gentle arc — no random jumping                   |
| Real GPS source                | Uses Fused Location Provider when configured                           |
| Full Codec 8 IO                | Ignition, DIN/DOUT, GSM, accelerometer, voltages, odometer, sleep mode |
| Stop Fleet w/ ignition OFF     | Sends a final ignition-0 frame per device on stop                      |
| Foreground service + wake lock | Keeps sending while app is backgrounded or screen locked               |
| Battery-optimization exemption | Requests Doze whitelist so routes keep streaming                       |
| Route preview map              | osmdroid polyline preview, no API key required                         |

### Mobile App (Capacitor, F-12)

* Biometric auth (fingerprint / face) on app start
* Encrypted credential storage (SecureStorage)
* Same React web UI wrapped in a Capacitor shell

### Centralized Logging (F-13)

* Serilog to Console (stdout) **and** Seq (structured logs)
* Seq web UI at port 8083; logs shipped via `Seq__Url` env var

### Dashboard (F-14) & Settings (F-15)

* Personalized dashboard with device stats and recent geofence events
* Settings: alert thresholds, notification preferences, company offline threshold, trip ignition gap

***
