# Navon Server (C# / .NET 9)

Self-hosted GPS tracking backend for **Teltonika FMB series** devices (Codec 8 / Codec 8 Extended).

Single deployable ASP.NET Core service, built vertical-slice/CQRS style with MediatR, storing
telemetry as native PostGIS geography points. No auth/multi-tenancy layer — everything is
single-tenant by design for this scope. (If you outgrow that later, see the note at the bottom.)

## Architecture

```
FMB Device ──(TCP 5027)──► TeltonikaListener (BackgroundService)
                                    │
                            Codec 8/8E Parser
                                    │
                        ProcessGpsTelemetry (MediatR command)
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                    │
         Postgres+PostGIS   GeofenceCheckService   WebhookDispatchService
         (Dapper, geography)        │                    │
                │              LiveTrackingHub       HTTP POST (HMAC-signed)
           REST API (/api)      (SignalR)             → your endpoint
```

Everything above runs inside one process (`MyGpsTracker.Api`) so device ingestion never blocks
HTTP requests and vice versa — the TCP listener is a `BackgroundService` running alongside Kestrel.

## Quick Start

```bash
cd GPSTracker
docker-compose up --build
```

| Service | URL |
|---------|-----|
| Swagger API | http://localhost:8080/swagger |
| SignalR Hub | ws://localhost:8080/hubs/tracking |
| TCP (devices) | :5027 |
| Postgres | localhost:5432 (db: `gpstracker` / user: `gpstracker` / pass: `gpstracker`) |

No separate migration step — `db/init.sql` runs automatically the first time the `db` container
starts (creates the `postgis` extension and all tables).

### Running without Docker

```bash
# 1. Start Postgres+PostGIS however you like, then:
psql "postgresql://gpstracker:gpstracker@localhost:5432/gpstracker" -f db/init.sql

# 2. Run the API
cd api/MyGpsTracker.Api
dotnet restore
dotnet run
```

> **Note on .NET version:** the `.csproj` targets `net9.0` (current LTS-adjacent stable at time of
> writing). Bump `<TargetFramework>` to `net10.0` once you have the .NET 10 SDK installed locally —
> nothing else needs to change.

## Configure Your FMB Device (Teltonika Configurator)

| Setting | Value |
|---------|-------|
| Server Address | your-server-ip |
| Server Port | 5027 |
| Protocol | TCP |
| Codec | Codec 8 (or Codec 8 Extended) |

## REST API

### Devices
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/devices` | List all devices |
| GET | `/api/devices/{imei}` | Device info |
| PATCH | `/api/devices/{imei}` | Update display name |
| GET | `/api/devices/{imei}/locations` | History (`?from=&to=&limit=`) |
| GET | `/api/devices/{imei}/locations/latest` | Last known position |
| GET | `/api/devices/{imei}/trips` | Trips derived from ignition on/off transitions |

### Geofences
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/geofences` | List geofences |
| POST | `/api/geofences` | Create geofence (circle or polygon) |
| DELETE | `/api/geofences/{id}` | Delete geofence |
| GET | `/api/geofences/events` | Crossing events (`?geofenceId=&imei=&limit=`) |

Circle geofence body:
```json
{ "name": "Warehouse", "type": 0, "centerLatitude": 9.0192, "centerLongitude": 38.7525, "radiusMeters": 150 }
```

Polygon geofence body:
```json
{ "name": "Depot Yard", "type": 1, "polygonWkt": "POLYGON((38.75 9.01, 38.76 9.01, 38.76 9.02, 38.75 9.02, 38.75 9.01))" }
```

### Webhooks
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/webhooks` | List webhooks |
| POST | `/api/webhooks` | Register endpoint |
| DELETE | `/api/webhooks/{id}` | Remove webhook |
| PATCH | `/api/webhooks/{id}/toggle` | Enable/disable |

## Webhook Payload Examples

**Location event:**
```json
{
  "event_type": "location",
  "imei": "352093081234567",
  "timestamp": "2026-01-15T10:30:00Z",
  "latitude": 9.0192,
  "longitude": 38.7525,
  "speed": 45,
  "ignition": true
}
```

**Geofence event:**
```json
{
  "event_type": "geofence_enter",
  "imei": "352093081234567",
  "geofence_name": "Warehouse",
  "timestamp": "2026-01-15T10:35:00Z"
}
```

**Webhook security** — set a `secret` when registering and verify the `X-GPS-Signature: sha256=<hmac>`
header (HMAC-SHA256 over the raw JSON body, hex-encoded) on your endpoint.

## SignalR (Real-time)

```javascript
const conn = new signalR.HubConnectionBuilder()
    .withUrl("http://your-server:8080/hubs/tracking")
    .withAutomaticReconnect()
    .build();

await conn.start();
await conn.invoke("Subscribe", "352093081234567");

conn.on("LocationUpdate", (u) => console.log(u));
conn.on("GeofenceEvent",  (e) => console.log(e));
conn.on("IgnitionEvent",  (e) => console.log(e));
```

## Project Structure

```
GPSTracker/
├── db/
│   └── init.sql                    PostGIS extension + schema
├── api/
│   ├── MyGpsTracker.sln
│   └── MyGpsTracker.Api/
│       ├── Domain/                 Device, VehicleTelemetry, Geofence, WebhookConfig
│       ├── Ingestion/
│       │   ├── Protocols/          Codec8Parser, IoElementIds, IProtocolParser
│       │   └── Sockets/            TeltonikaListener (TCP BackgroundService)
│       ├── Features/                Vertical slices (MediatR commands/queries + handlers)
│       │   ├── Telemetry/          ProcessGpsTelemetry — the central ingestion pipeline
│       │   ├── Devices/            List/get/update/history/trips
│       │   ├── Geofences/          Create/list/delete/events + Haversine/point-in-polygon check
│       │   ├── Webhooks/           Register/list/delete/toggle + HMAC dispatch
│       │   └── Tracking/           LiveTrackingHub (SignalR)
│       ├── Api/Controllers/        Thin controllers that just call mediator.Send(...)
│       ├── Infrastructure/         Npgsql DI registration
│       ├── Program.cs
│       └── MyGpsTracker.Api.csproj
├── docker-compose.yml
├── web/
│   └── ...
└── gps-simulator/
    └── ...
```

## Scoped down from the original design

This build intentionally **drops** the multi-tenant / Keycloak layer described in the original
enterprise design doc (per-client OAuth, `client_vehicle_rights` ACLs, tenant-scoped SignalR
groups) to keep this a single-tenant, self-hosted service. If you need that later:

- Add a `clients` table + `client_vehicle_rights` join table.
- Put Keycloak (or any OIDC provider) in front of the API, add `[Authorize]` + a `ClaimsPrincipal`-based
  tenant resolver.
- Scope every query in `Features/*` by the caller's tenant/client ID, and scope `LiveTrackingHub`
  groups by tenant instead of just IMEI.

None of the vertical-slice/CQRS structure needs to change to add that — it's additive.
