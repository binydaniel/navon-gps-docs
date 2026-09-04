# Software Requirements Specification (SRS)

## GPSTracker - Fleet Management & Telemetry Platform

**Version:** 1.0  
**Date:** August 2026  
**Repository:** github.com/binydaniel/MyGpsTracker (private) 

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Technology Stack](#3-technology-stack)
4. [Architecture](#4-architecture)
5. [Detailed Feature Specifications](#5-detailed-feature-specifications)
6. [API Reference](#6-api-reference)
7. [Database Schema](#7-database-schema)
8. [Deployment & Infrastructure](#8-deployment--infrastructure)
9. [Security](#9-security)
10. [Non-Functional Requirements](#10-non-functional-requirements)
11. [Budget & Cost Estimate](#11-budget--cost-estimate)
12. [Implementation Roadmap](#12-implementation-roadmap)
13. [Operational Workflow](#13-operational-workflow)

---

## 1. Introduction

### 1.1 Purpose

GPSTracker is a real-time fleet management and telemetry platform that ingests GPS data from physical tracking devices (Teltonika FMB series, Queclink, GT06), processes telemetry in real-time, and presents fleet intelligence through a web dashboard, Android mobile app, and a standalone GPS simulator for testing.

### 1.2 Scope

The system supports:

- Multi-tenant fleet management (companies, users, drivers, vehicles)
- Real-time vehicle tracking with live map
- Geofencing with enter/exit detection, violations, and per-zone speed limits
- Remote device configuration and command execution (Codec 12)
- Comprehensive event detection: eco scoring, idling, jamming, movement, speeding
- Notification engine with configurable thresholds and real-time push
- Webhook integrations for external system connectivity
- Biometric authentication on mobile
- Centralized logging (Seq/Serilog)
- GPS device simulation for development and testing

### 1.3 Target Users

| User Role | Description |
|-----------|-------------|
| System Administrator | Global superuser - manages companies, global settings |
| Company Admin | Manages users, devices, vehicles, geofences, webhooks within their company |
| Company User | Read-only view of assigned vehicles on the live map |
| Driver | Read-only view of their own assigned vehicle(s) |
| Fleet Manager | Monitors fleet via dashboard, map, notifications, and reports |
| External Systems | Consume webhook events (ERP, dispatch, insurance) |

---

## 2. System Overview

### 2.1 High-Level Architecture

```
                         CLIENTS
 +----------+  +------------+  +----------+  +----------+
 | Web App  |  | Android    |  | GPS Sim  |  | Webhook  |
 | (React)  |  | (Capacitor)|  | (Standalone| | Consum. |
 |          |  | Hybrid)    |  | Android) |  |          |
 +----+-----+  +-----+------+  +----+-----+  +----+-----+
      |              |               |               |
      +-- REST/HTTP + SignalR (WS) -+               |
                   |         TCP Codec 8/12          |
                   +---------+---------+             |
                             |         |             |
                   +---------+---------+-------------+
                   |     API SERVER                   |
                   |  ASP.NET Core 9 / C#            |
                   |  MediatR + Dapper + SignalR     |
                   +--+----------+----------+--------+
                      |          |          |
             +--------+--+ +----+----+ +---+-------+
             | PostgreSQL | | Seq     | | nginx     |
             | + PostGIS  | |         | | (proxy)   |
             |            | |         | |           |
             +------------+ +---------+ +-----------+
```

### 2.2 Key Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| API Server | ASP.NET Core 9 | REST API + SignalR hub + TCP device ingestion |
| Web Frontend | React 18 + Vite + TailwindCSS | Dashboard, live map, admin UI |
| PostgreSQL + PostGIS | PostGIS 16-3.4 | Spatial database with geofence queries |
| Seq | Seq (Datalust) | Centralized structured logging UI |
| GPS Simulator | Kotlin/Android | Simulates Teltonika devices for testing |
| Mobile App | Capacitor (Android) | Biometric-authenticated mobile access |

### 2.3 Docker Services

| Service | Image | Port Mapping | Volume |
|---------|-------|-------------|--------|
| db | postgis/postgis:16-3.4 | 5433:5432 | gpstracker-db-data |
| api | Custom build | 8081:8080, 5027:5027, 5011:5011, 5023:5023 | - |
| seq | datalust/seq:latest | 8083:80, 5341:5341 | seq-data |
| web | Custom build (nginx) | 8082:80 | - |

---

## 3. Technology Stack

### 3.1 Backend

| Technology | Version | Purpose |
|-----------|---------|---------|
| .NET | 9.0 | Runtime |
| ASP.NET Core | 9.0.9 | Web framework, routing, middleware |
| Entity Framework Core | 9.0.9 | ASP.NET Core Identity tables only |
| Dapper | 2.1.35 | High-performance data access for CQRS |
| MediatR | 12.4.1 | In-process command/query mediator |
| Npgsql | 9.0.3 | PostgreSQL ADO.NET driver |
| Serilog.AspNetCore | 8.0.3 | Structured logging |
| Serilog.Sinks.Seq | 9.0.0 | Log shipping to Seq |
| JWT Bearer Auth | 9.0.9 | Stateless API authentication |
| Swashbuckle | 6.7.3 | Swagger/OpenAPI documentation |
| SignalR | Built-in | Real-time WebSocket push |

### 3.2 Frontend

| Technology | Version | Purpose |
|-----------|---------|---------|
| React | 18.3.1 | UI framework |
| TypeScript | 5.6.2 | Static type checking |
| Vite | 5.4.8 | Build tool, HMR |
| TailwindCSS | 3.4.13 | Utility-first CSS |
| React Router | 6.26.2 | Client-side hash routing |
| TanStack React Query | 5.59.16 | Server state, caching, mutations |
| Leaflet + React-Leaflet | 1.9.4 / 4.2.1 | Map rendering |
| @microsoft/signalr | 8.0.7 | Real-time WebSocket client |
| Capacitor | 8.5.0 | Android hybrid wrapper |

### 3.3 GPS Simulator App (Standalone)

| Technology | Purpose |
|-----------|---------|
| Kotlin | Language |
| Android SDK | Platform (minSdk 26) |
| java.net.Socket | TCP communication |
| Fused Location Provider | Real GPS hardware location |
| Foreground Service | Persistent sending while app is backgrounded |
| WakeLock API | Prevent CPU sleep during sending |

### 3.4 Mobile App (Capacitor Hybrid)

| Technology | Purpose |
|-----------|---------|
| @aparajita/capacitor-biometric-auth | Fingerprint / face login |
| capacitor-secure-storage-plugin | Encrypted credential storage |
| MainActivity (Java) | Plugin registration |
| Same React web UI | Wrapped in Capacitor shell |

---

## 4. Architecture

### 4.1 Backend: CQRS with MediatR + Dapper

- **Commands** (writes): MediatR IRequest handlers, raw Dapper SQL, manual NpgsqlTransaction
- **Queries** (reads): MediatR handlers, Dapper SQL, optimized PostgreSQL indexes
- **EF Core** reserved only for ASP.NET Core Identity (AspNetUsers, AspNetRoles, AspNetUserRoles)
- **No controllers contain business logic** - all logic in feature handlers

### 4.2 Telemetry Ingestion Pipeline

```
Physical Device (TCP connection)
  -> DeviceTcpListener.AcceptClientAsync()
    -> Protocol Handshake (IMEI identification)
    -> DeviceConnectionRegistry.Register(imei, stream)
    -> Frame Loop:
      -> IDeviceFramer.ReadFrameAsync()      -- TCP frame delineation
      -> IProtocolParser.TryParse()           -- binary decode
      -> ITelemetryNormalizer.Normalize()     -- IO element mapping
      -> ProcessGpsTelemetry.Command          -- MediatR dispatch
        1. Persist to vehicle_telemetry (PostGIS geography POINT)
        2. Push LocationUpdate via SignalR
        3. IgnitionEvent (if state changed)
        4. Geofence evaluation
        5. Per-geofence speeding check
        6. Driving event detection
        7. Webhook dispatch
        8. Notification evaluation
    -> SendAckAsync (Codec 8 record count)
```

### 4.3 SignalR Real-Time Hub

**Hub class:** `LiveTrackingHub`

**Groups:**
- `device-{imei}` - subscribers tracking a specific device
- `company-{companyId}` - all users in a company (notifications)

**Messages (Server to Client):**

| Message | Trigger |
|---------|---------|
| LocationUpdate | Every telemetry point (lat, lng, speed, ignition, heading, DIO, accel, odometer, etc.) |
| IgnitionEvent | Ignition state changes |
| GeofenceEvent | Enter/exit geofence |
| GeofenceViolation | Vehicle exits assigned geofence |
| GeofenceViolationResolved | Vehicle re-enters geofence |
| Notification | Alert threshold exceeded |
| NewDeviceRegistered | Unknown device connects |

### 4.4 Protocol Abstraction

```
IDeviceProtocol
  +-- Name: string              ("teltonika", "queclink", "gt06")
  +-- CreateFramer()      -> IDeviceFramer      (TCP frame delineation)
  +-- CreateHandshake()   -> IDeviceHandshake   (IMEI identification)
  +-- CreateParser()      -> IProtocolParser    (bytes -> ParsedFrame)
  +-- CreateNormalizer()  -> ITelemetryNormalizer (IO -> NormalizedTelemetry)
```

### 4.5 Key Backend Components

| Component | File | Purpose |
|-----------|------|---------|
| DeviceTcpListener | Ingestion/Sockets/DeviceTcpListener.cs | Accepts TCP connections, runs protocol loop |
| DeviceConnectionRegistry | Ingestion/Sockets/DeviceConnectionRegistry.cs | In-memory IMEI->Stream lookup for Codec 12 |
| Codec12CommandBuilder | Ingestion/Teltonika/Codec12CommandBuilder.cs | Builds binary Codec 12 frames with CRC32 |
| TeltonikaNormalizer | Ingestion/Teltonika/TeltonikaNormalizer.cs | Maps 17+ IO elements to NormalizedTelemetry |
| IoElementIds | Ingestion/Teltonika/IoElementIds.cs | Constants for all Teltonika FMB IO element IDs |
| ProcessGpsTelemetry | Features/Telemetry/ProcessGpsTelemetry.cs | Main telemetry pipeline |
| DeviceIgnitionStateCache | Features/Telemetry/DeviceIgnitionStateCache.cs | Tracks ignition state for change detection |
| GeofenceCheckService | Features/Geofences/GeofenceCheckService.cs | Real-time geofence evaluation |
| GeofenceStateCache | Features/Geofences/GeofenceStateCache.cs | Caches inside/outside state per device-geofence |
| NotificationEvaluatorService | Features/Notifications/NotificationEvaluatorService.cs | Threshold checks with cooldown |
| WebhookDispatchService | Features/Webhooks/WebhookDispatchService.cs | Outbound HTTP webhook fan-out |

---

## 5. Detailed Feature Specifications

### F-01: Authentication & Authorization

**Backend:** `Features/Auth/` + `Api/Controllers/AuthController.cs`  
**Frontend:** `features/auth/` (LoginPage.tsx, RegisterPage.tsx, AuthContext.tsx)

| Sub-Feature | Description |
|------------|-------------|
| User Registration | Email + password, creates AspNetUsers row via ASP.NET Core Identity |
| Login | Credentials validated, signed JWT returned with roles, companyId, email |
| JWT Token | Contains sub (user ID), email, role array, company_id; validated on every request |
| Password Change | Requires current password, server-side validation |
| Role System | 4 roles: SystemAdmin, Admin, User, Driver |
| Multi-Tenant Scoping | Data filtered by company_id on all queries |
| Biometric Auth (Mobile) | Capacitor biometric plugin + SecureStorage for credential caching |
| Session Persistence | JWT stored in localStorage; auto-refresh on page load via /me endpoint |
| Credential Caching | On login, credentials saved to SecureStorage; on logout, cleared |

**Role Permissions Matrix:**

| Role | Scope | Can See | Can Manage |
|------|-------|---------|-----------|
| SystemAdmin | Global (no company) | All companies, all devices | Companies, global settings |
| Admin | Company-scoped | All company devices/users | Users, vehicles, geofences, webhooks, device assignment |
| User | Company-scoped | Assigned vehicles only | Nothing (read-only) |
| Driver | Company-scoped | Assigned vehicles only | Nothing (read-only) |

### F-02: Device Management

**Backend:** `Features/Devices/DeviceFeatures.cs`, `DeviceAccess.cs`  
**Frontend:** `features/devices/DevicesPage.tsx`, `DeviceDetailPage.tsx`

| Sub-Feature | Description |
|------------|-------------|
| Auto-Registration | Devices register on first telemetry connection (RegisterDevice command) |
| Device List | Admin sees all company devices (via company_devices junction); User/Driver sees assigned only |
| Device Detail | IMEI, model, protocol, phone number, codec version, offline threshold, created date |
| Device History | Position history with polyline replay on map, time range filtering |
| Trip History | Trips derived from ignition transitions with distance, max speed, start/end times |
| Device Events | Eco scoring, idling, jamming, movement events displayed in table |
| Power Log | 24h voltage/battery history table |
| Geofence Violations | Violations for the device displayed on detail page |
| Device Search | Search/filter devices by name or IMEI |
| Edit Device | Admin can update display name, offline threshold |
| Delete Device | Admin can remove device (cascades to vehicle_telemetry, device_events) |
| Multi-Company Assignment | company_devices junction table: one device visible to multiple companies |
| Vehicle Assignment | Device to Vehicle 1:1 unique mapping (unique index on vehicles.device_id) |
| SIM / Phone Number | Admin can set/edit phone number for SMS from device detail page |
| SMS Integration | sms:{number} link for sending SMS to device SIM card |

### F-03: Live Vehicle Tracking

**Frontend:** `features/tracking/LiveMapPage.tsx`, `LiveMap.tsx`, `DeviceMarker.tsx`  
**Backend:** `Features/Tracking/LiveTrackingHub.cs`

| Sub-Feature | Description |
|------------|-------------|
| Real-Time Map | Leaflet map with live-updating device markers via SignalR |
| Marker Colors | Green = external power, Yellow = battery, Gray = offline/not reporting |
| Battery Badge | Yellow battery icon on marker when on battery power |
| Ignition Badge | ON/OFF indicator in marker tooltip and sidebar list |
| Device Sidebar | Scrollable list of all devices with name, status, speed, last update |
| Device Selection | Click device on map or sidebar to inspect details |
| Offline Detection | Configurable threshold (default 10 min per company) |
| Position Merging | SignalR live position wins; REST latest location as fallback |
| Telemetry Detail | DIN 1-4, DOUT 1-3, accelerometer XYZ, GSM signal, odometer, sleep mode, movement |

### F-04: Geofencing

**Backend:** `Features/Geofences/GeofenceFeatures.cs`, `GeofenceCheckService.cs`, `GeofenceStateCache.cs`  
**Frontend:** `features/geofences/GeofencesPage.tsx`

| Sub-Feature | Description |
|------------|-------------|
| Circle Geofence | Center (lat/lng) + radius in meters |
| Polygon Geofence | WKT polygon string stored in polygon_wkt |
| Geofence-Vehicle Mapping | Many-to-many via vehicle_geofences junction table |
| Enter/Exit Detection | Real-time on every telemetry point via GeofenceCheckService |
| Violation Tracking | geofence_violations table: enter time, resolved time, location |
| Active Violations | Live list of vehicles currently outside assigned geofences |
| Speed Limit per Zone | speed_limit_kph on geofences - speeding generates notification |
| Geofence Events | Recent enter/exit events displayed on dashboard |
| PostGIS ST_Contains | Spatial query for polygon containment |
| Haversine Distance | Circle containment via distance calculation |
| State Cache | In-memory cache avoids redundant DB queries for same device-geofence pair |

### F-05: Remote Device Configuration (Codec 12)

**Backend:** `Features/Devices/DeviceCommandFeatures.cs` + `Ingestion/Teltonika/Codec12CommandBuilder.cs`  
**Frontend:** `features/devices/DeviceDetailPage.tsx` (Remote Commands card)

| Sub-Feature | Description |
|------------|-------------|
| Send Arbitrary Command | Text input: setparam, getinfo, getver, etc. |
| Quick Buttons | One-click: getinfo, getver, getstatus, ggps, flush, setparam APN |
| CPU Reset | Confirmation dialog before sending cpureset |
| Engine Immobilizer | setdigout 1 (activate) / setdigout 0 (deactivate) |
| Codec 12 Frame Builder | Binary protocol: 0x00 + length(2B) + ASCII command + CRC32 |
| Connection Registry | In-memory ConcurrentDictionary with WeakReference to TCP streams |
| Command Delivery | Finds live TCP stream for device IMEI, writes Codec 12 frame |
| Online Indicator | Commands only work when device is connected (port 5027) |

### F-06: Notification Engine

**Backend:** `Features/Notifications/NotificationEvaluatorService.cs`, `NotificationFeatures.cs`  
**Frontend:** `features/notifications/` (NotificationBanner.tsx), `features/settings/SettingsPage.tsx`

| Sub-Feature | Description |
|------------|-------------|
| Threshold-Based Alerts | Configurable per company, per alert type |
| Alert Types | speeding, hard_brake, hard_curve, low_battery, offline, geofence_speeding |
| Configurable Thresholds | Speed (km/h), G-force (brake/curve), voltage (battery) |
| 2-Minute Cooldown | Prevents alert flooding for same device+type |
| Delta Detection | Heading delta (>30 degrees), speed delta (>20 km/h) between readings |
| Real-Time Push | SignalR Notification message to company group |
| Notification Persistence | Stored in notifications table with severity, read status |
| Unread Count | Badge count on notification bell |
| Mobile Push Toast | In-app banner for new alerts |
| Enable/Disable per Type | Admin can toggle each alert type independently |

### F-07: Eco Scoring & Driving Events

**Backend:** `Features/Telemetry/ProcessGpsTelemetry.cs` (DetectDrivingEventsAsync)

| Event Type | Detection Logic | IO Elements Used |
|-----------|----------------|-----------------|
| Harsh Acceleration | abs_acceleration > 400 (4.0 m/s2) | ID 23 |
| Hard Braking | Speed delta > 20 km/h between consecutive readings | Speed field |
| Hard Cornering | Heading delta > 30 degrees between consecutive readings | Angle field |
| Idling | Speed=0 + Ignition=ON for >5 minutes | Speed, Ignition |
| Movement While Off | Movement IO=1 + Ignition=OFF | ID 240 |
| Signal Jamming | GNSS status=0 while Ignition=ON | ID 69 |
| Low GSM Signal | GSM signal <= 5 while Ignition=ON | ID 21 |

All events are persisted to device_events table with severity, lat/lng, and JSON data payload.

### F-08: Per-Geofence Speeding

**Backend:** `Features/Telemetry/ProcessGpsTelemetry.cs` (CheckGeofenceSpeedingAsync)

| Sub-Feature | Description |
|------------|-------------|
| Speed Limit per Geofence | Admin sets speed_limit_kph when creating/editing geofence |
| Real-Time Check | Every telemetry point checked against assigned geofence speed limits |
| Haversine Distance | Distance from vehicle to geofence center calculated |
| Threshold | Alert triggers when speed exceeds limit + 2 km/h buffer |
| Notification | Geofence speeding notification with zone name, vehicle speed, limit |
| Severity | Warning level |

### F-09: Power Monitoring

**Backend:** ProcessGpsTelemetry computes power source from voltages  
**Frontend:** `DeviceDetailPage.tsx` (Power Log table)

| Sub-Feature | Description |
|------------|-------------|
| External Voltage | IO element 66, stored as voltage (raw/1000) |
| Battery Voltage | IO element 67, stored as voltage (raw/1000) |
| Power Source Computation | external if >6.0V, battery if >2.0V, else off |
| Power Log API | GET /api/devices/{id}/power-log - 24h history |
| Power Log UI | Table showing timestamp, external V, battery V, source |
| Marker Color | Green=external, Yellow=battery, Gray=off |

### F-10: Digital I/O Monitoring

**Backend:** TeltonikaNormalizer maps DIN/DOUT IO elements  
**Frontend:** DeviceDetailPage.tsx (Telemetry tab)

| Sub-Feature | Description |
|------------|-------------|
| DIN 1-4 | Digital inputs: IO IDs 1, 2, 3, 247 |
| DOUT 1-3 | Digital outputs: IO IDs 33, 34, 35 |
| Live via SignalR | All DIO values pushed with every LocationUpdate |
| Stored per Reading | In vehicle_telemetry table columns digital_input_1-4, digital_output_1-3 |
| Immobilizer Control | DOUT 1 used for engine immobilizer via Codec 12 setdigout command |

### F-11: Webhooks

**Backend:** `Features/Webhooks/WebhookFeatures.cs`, `WebhookDispatchService.cs`  
**Frontend:** `features/webhooks/WebhooksPage.tsx`

| Sub-Feature | Description |
|------------|-------------|
| Webhook Registration | Admin creates webhook with URL, secret, optional event type filter |
| Event Types | location, geofence_enter, geofence_exit, new_device |
| HMAC Signing | Webhook payload signed with configured secret |
| Fire-and-Forget | Webhooks dispatched asynchronously, non-blocking |
| Retry | No retry (fire-and-forget design) |
| Admin Management | Create, enable/disable, delete webhooks |

### F-12: GPS Simulator App (Standalone Android)

**Directory:** `gps-simulator/`  
**Language:** Kotlin  
**Protocol:** Teltonika Codec 8 (binary TCP to port 5027)

| Sub-Feature | Description |
|------------|-------------|
| IMEI Handshake | Generates unique IMEI (SHA-256 of Android ID + Luhn check) |
| Real GPS | Uses Fused Location Provider for real device location |
| Foreground Service | Continues sending when app is backgrounded |
| WakeLock | Prevents CPU sleep during active session |
| Configurable Interval | 1-60 second send interval |
| SendConfig | All 17+ IO elements configurable |
| DIN 1-4 Toggles | Switch controls for digital inputs |
| DOUT 1-3 Toggles | Switch controls for digital outputs |
| Ignition Toggle | Mandatory ON to start sending |
| Jamming Toggle | Simulates GNSS jamming (GnssStatus=0) |
| GSM Signal Input | Configurable signal strength (0-31) |
| Accelerometer | Configurable X/Y/Z acceleration values |
| Voltage Settings | External and battery voltage |
| Sleep Mode | Sleep mode IO element |
| Odometer | Simulated total odometer |
| Log Viewer | Real-time packet log with scrolling |
| Packet Counter | Displays packets sent count |
| Connection Status | Color-coded status indicator |

**Codec 8 IO Elements Supported:**

| IO Element | ID | Description |
|-----------|-----|-------------|
| Digital Input 1 | 1 | DIN 1 |
| Digital Input 2 | 2 | DIN 2 |
| Digital Input 3 | 3 | DIN 3 |
| Accelerometer X | 17 | X-axis acceleration |
| Accelerometer Y | 18 | Y-axis acceleration |
| Accelerometer Z | 19 | Z-axis acceleration |
| Absolute Acceleration | 23 | Total acceleration magnitude |
| Total Odometer | 16 | Odometer in meters |
| GSM Signal | 21 | Signal strength (0-31) |
| Sleep Mode | 200 | Device sleep state |
| Digital Output 1 | 33 | DOUT 1 |
| Digital Output 2 | 34 | DOUT 2 |
| Digital Output 3 | 35 | DOUT 3 |
| Ignition | 239 | Ignition state |
| Movement | 240 | Movement detection |
| Digital Input 4 | 247 | DIN 4 |
| GNSS Status | 69 | 0=jamming, 1-7=fix quality |

### F-13: Mobile Capacitor App

**Directory:** `web/android/`  
**Framework:** Capacitor 8.5 (Hybrid)

| Sub-Feature | Description |
|------------|-------------|
| Biometric Auth | Fingerprint / face login on app start |
| Credential Caching | Saved to SecureStorage on login, auto-loaded on next launch |
| Auto-Biometric | If credentials cached, biometric prompt shown immediately |
| Manual Login | Fallback to email/password form |
| "Sign in with biometrics" | Explicit button for biometric auth |
| Same Web UI | All web features available in mobile shell |
| MainActivity Plugins | BiometricAuthNative + SecureStoragePluginPlugin registered |

### F-14: Centralized Logging

**Backend:** `Program.cs` (Serilog via `UseSerilog()` + `appsettings.json` Serilog section)  
**Infrastructure:** Seq (Datalust)

| Sub-Feature | Description |
|------------|-------------|
| Serilog Integration | ASP.NET Core request logging via `UseSerilog()` |
| Seq Sink | Ships all logs to Seq via `appsettings.json` Serilog config |
| Seq Web UI | Accessible via internal port (not exposed publicly in production) |
| No Authentication | SEQ_FIRSTRUN_NOAUTHENTICATION=true for development |
| Persistent Storage | Seq data volume mounted |

### F-15: Dashboard

**Frontend:** `features/dashboard/DashboardPage.tsx`

| Sub-Feature | Description |
|------------|-------------|
| Welcome Message | Personalized greeting with company name |
| Device Statistics | Total devices, online now, offline count |
| Online Detection | Devices reporting within last 10 minutes considered online |
| Quick Links | Live Map, Devices, Admin (if Admin), Companies (if SystemAdmin) |
| Geofence Events | Recent 8 geofence enter/exit events |
| Event Table | Device name, event type (Enter/Exit), geofence name, timestamp |
| Empty State | Friendly message when no devices or events |

### F-16: Settings

**Frontend:** `features/settings/SettingsPage.tsx`

| Sub-Feature | Description |
|------------|-------------|
| Thresholds Tab | Configure alert threshold values per alert type |
| Notification Preferences | Enable/disable notifications per alert type |
| Company Offline Threshold | Minutes before device shown as offline |
| Trip Ignition Gap | Minutes for trip segmentation |
| Admin Only | Settings page restricted to Admin and SystemAdmin roles |

### F-17: Multi-Company Device Assignment

**Backend:** `Features/Devices/DeviceAccess.cs`  
**Database:** `company_devices` junction table

| Sub-Feature | Description |
|------------|-------------|
| Junction Table | company_devices (company_id, device_id) - many-to-many |
| Admin View | Sees all devices assigned to their company |
| User/Driver View | Sees only devices linked to their assigned vehicles |
| Device Access Layer | DeviceAccess.cs computes visible device IDs based on role |
| Multi-Tenant | One device can be visible to multiple companies simultaneously |

---

## 6. API Reference

### 6.1 Authentication Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| POST | /api/auth/register | Anonymous | Register new user |
| POST | /api/auth/login | Anonymous | Login, returns JWT |
| POST | /api/auth/change-password | JWT | Change password |
| GET | /api/auth/me | JWT | Get current user profile |

### 6.2 Device Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/devices | JWT | List devices (filtered by role) |
| GET | /api/devices/{id} | JWT | Get device by ID |
| PATCH | /api/devices/{id} | JWT | Update device display name |
| DELETE | /api/devices/{id} | JWT | Delete device (cascades telemetry) |
| GET | /api/devices/{id}/locations | JWT | Get position history (from/to/limit) |
| GET | /api/devices/{id}/locations/latest | JWT | Get latest position |
| GET | /api/devices/{id}/trips | JWT | Get trip history from ignition transitions |
| PATCH | /api/devices/{id}/phone | Staff | Set SIM phone number |

### 6.3 Device Command Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/devices/connections | SystemAdmin | List connected device IMEIs |
| POST | /api/devices/{id}/command | Admin | Send Codec 12 command |
| POST | /api/devices/{id}/digout | Admin | Set digital output |
| POST | /api/devices/{id}/param | Admin | Set device parameter |
| POST | /api/devices/{id}/reset | Admin | CPU reset |
| GET | /api/devices/{id}/events | JWT | Get device events |
| GET | /api/devices/{id}/power-log | JWT | Get power voltage log |

### 6.4 Geofence Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/geofences | Admin | List geofences |
| POST | /api/geofences | Admin | Create geofence (circle/polygon) |
| PUT | /api/geofences/{id} | Admin | Update geofence |
| DELETE | /api/geofences/{id} | Admin | Delete geofence |
| GET | /api/geofences/events | Admin | Get enter/exit events |
| GET | /api/geofences/{id}/vehicles | Admin | List assigned vehicles |
| POST | /api/geofences/{id}/vehicles/{vehicleId} | Admin | Assign vehicle |
| DELETE | /api/geofences/{id}/vehicles/{vehicleId} | Admin | Remove vehicle |
| GET | /api/geofences/{id}/violations | Admin | Get violations |
| GET | /api/vehicles/{vehicleId}/geofence-violations | Admin | Get violations for a vehicle |

### 6.5 Notification Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/notifications | JWT | List notifications |
| GET | /api/notifications/unread-count | JWT | Unread count |
| PUT | /api/notifications/{id}/read | JWT | Mark as read |
| PUT | /api/notifications/read-all | JWT | Mark all as read |
| DELETE | /api/notifications/{id} | JWT | Dismiss notification |
| GET | /api/notifications/settings | Staff | Get notification settings |
| POST | /api/notifications/settings | Staff | Upsert notification setting |
| POST | /api/notifications/settings/bulk | Staff | Bulk upsert all settings |

### 6.6 Admin Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/admin/users | Staff | List company users |
| POST | /api/admin/users | Staff | Create user |
| POST | /api/admin/users/link | Staff | Link self-registered user |
| PUT | /api/admin/users/{id}/vehicles | Staff | Assign vehicles to user |
| GET | /api/admin/users/search | Staff | Search users by email |
| PUT | /api/admin/users/{id}/role | Staff | Set user role |
| DELETE | /api/admin/users/{id} | Staff | Unlink user from company |
| GET | /api/admin/devices | Staff | List company devices |
| POST | /api/admin/devices | SystemAdmin | Add existing device to company |
| POST | /api/admin/devices/create | SystemAdmin | Create device + assign to company |
| PUT | /api/admin/devices/{id}/phone | Staff | Set SIM phone number |
| PUT | /api/admin/devices/{id}/offline-threshold | Staff | Set device offline threshold |
| GET | /api/admin/vehicles | Staff | List company vehicles |
| POST | /api/admin/vehicles | Staff | Create vehicle |
| POST | /api/admin/vehicles/{id}/device | Staff | Link device to vehicle |
| DELETE | /api/admin/vehicles/{id} | Staff | Delete vehicle |
| PUT | /api/admin/settings/offline-threshold | Staff | Set company offline threshold |
| PUT | /api/admin/settings/trip-gap | Staff | Set company trip gap |

### 6.7 Company Endpoints (SystemAdmin only)

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/companies | SystemAdmin | List all companies |
| POST | /api/companies | SystemAdmin | Create company |
| PUT | /api/companies/{id} | SystemAdmin | Update company |
| POST | /api/companies/{id}/admins | SystemAdmin | Create admin for company |
| DELETE | /api/companies/{id} | SystemAdmin | Delete company (cascades) |

### 6.8 System Admin Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/admin/system/users | SystemAdmin | List all users across companies |
| PUT | /api/admin/system/users/{id}/company | SystemAdmin | Set/clear user company |
| DELETE | /api/admin/system/users/{id} | SystemAdmin | Delete user |
| PATCH | /api/admin/system/users/{id}/display-name | SystemAdmin | Update display name |
| GET | /api/admin/system/devices | SystemAdmin | List all devices |
| PUT | /api/admin/system/devices/{id}/company | SystemAdmin | Add/remove device from company |
| POST | /api/admin/system/devices | SystemAdmin | Register device by IMEI |
| GET | /api/admin/system/companies | SystemAdmin | List all companies |
| PUT | /api/admin/system/companies/{id}/active | SystemAdmin | Activate/deactivate company |
| GET | /api/admin/system/available-users | SystemAdmin | List unassigned users |
| POST | /api/admin/system/companies/{id}/admins | SystemAdmin | Assign user as company admin |
| DELETE | /api/admin/system/companies/{id} | SystemAdmin | Delete company (cascades) |

### 6.9 Webhook Endpoints

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | /api/webhooks | Staff | List webhooks |
| POST | /api/webhooks | Staff | Create webhook |
| DELETE | /api/webhooks/{id} | Staff | Delete webhook |
| PATCH | /api/webhooks/{id}/toggle | Staff | Enable/disable webhook |

### 6.10 TCP Device Ingestion

| Port | Protocol | Description |
|------|----------|-------------|
| 5027 | Teltonika Codec 8 | Primary device ingestion |
| 5011 | Queclink | Queclink device ingestion |
| 5023 | GT06 | GT06 device ingestion |

---

## 7. Database Schema

### 7.1 Core Tables

**devices** - GPS tracking devices

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| imei | varchar(32) UNIQUE | Device IMEI |
| display_name | varchar(128) | Human-readable name |
| protocol | varchar(32) DEFAULT 'teltonika' | Device protocol |
| model | varchar(64) | Device model |
| codec_version | smallint | Wire codec version |
| settings | jsonb | Per-device config |
| phone_number | varchar(32) | SIM card phone |
| offline_threshold_minutes | integer DEFAULT 10 | Minutes before offline |
| company_id | uuid FK | Owning company |
| created_at | timestamptz | Creation time |

**vehicle_telemetry** - GPS readings with telemetry

| Column | Type | Description |
|--------|------|-------------|
| id | bigserial PK | Auto-increment |
| device_id | uuid FK | Source device |
| timestamp | timestamptz | Reading time |
| location | geography(Point, 4326) | PostGIS point |
| speed_kph | double precision | Speed |
| ignition_on | boolean | Ignition state |
| satellites | integer | Satellite count |
| angle_degrees | integer | Heading |
| power_source | varchar(16) | external/battery/off |
| power_voltage | double precision | External voltage |
| battery_voltage | double precision | Battery voltage |
| gsm_signal | integer | GSM strength |
| digital_input_1-4 | integer | DIN values |
| digital_output_1-3 | integer | DOUT values |
| accel_x, accel_y, accel_z | integer | Accelerometer |
| abs_acceleration | integer | Total acceleration |
| total_odometer | bigint | Odometer (meters) |
| trip_odometer | bigint | Trip odometer |
| sleep_mode | integer | Sleep state |
| movement | integer | Movement detection |
| gnss_status | integer | GNSS status |
| raw_payload | bytea | Original frame |

**companies** - Tenant organizations

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| name | varchar(128) UNIQUE | Company name |
| is_active | boolean DEFAULT true | Active/inactive toggle |
| offline_threshold_minutes | integer DEFAULT 10 | Global threshold |
| trip_ignition_gap_minutes | integer DEFAULT 5 | Trip segmentation gap |
| created_at | timestamptz | Creation time |

**vehicles** - Fleet vehicles

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| company_id | uuid FK | Owning company |
| name | varchar(128) | Vehicle name |
| device_id | uuid FK UNIQUE | Assigned device (1:1) |
| created_at | timestamptz | Creation time |

### 7.2 Junction Tables

**company_devices** - Device-company (many-to-many)

| Column | Type | Description |
|--------|------|-------------|
| company_id | uuid FK | Company |
| device_id | uuid FK | Device |

**user_vehicles** - User-vehicle assignment

| Column | Type | Description |
|--------|------|-------------|
| user_id | uuid | AspNetUsers ID |
| vehicle_id | uuid FK | Vehicle |

**vehicle_geofences** - Vehicle-geofence assignment

| Column | Type | Description |
|--------|------|-------------|
| vehicle_id | uuid FK | Vehicle |
| geofence_id | uuid FK | Geofence |

### 7.3 Geofence Tables

**geofences** - Geofence zones

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| company_id | uuid FK | Owning company |
| name | varchar(128) | Zone name |
| type | integer | 0=Circle, 1=Polygon |
| center_latitude | double precision | Center lat |
| center_longitude | double precision | Center lng |
| radius_meters | double precision | Circle radius |
| polygon_wkt | text | WKT polygon |
| speed_limit_kph | double precision | Speed limit |
| created_at | timestamptz | Creation time |

**geofence_violations** - Active violations

| Column | Type | Description |
|--------|------|-------------|
| id | bigserial PK | Auto-increment |
| vehicle_id | uuid FK | Vehicle |
| geofence_id | uuid FK | Geofence |
| device_id | uuid FK | Device |
| entered_at | timestamptz | Violation start |
| resolved_at | timestamptz | Resolution time |
| latitude, longitude | double precision | Location |

**geofence_events** - Enter/exit events

| Column | Type | Description |
|--------|------|-------------|
| id | bigserial PK | Auto-increment |
| geofence_id | uuid FK | Geofence |
| device_id | uuid FK | Device |
| event_type | integer | 0=Enter, 1=Exit |
| timestamp | timestamptz | Event time |

### 7.4 Notification & Event Tables

**notification_settings** - Alert configuration

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| company_id | uuid FK | Company |
| alert_type | varchar(32) | Alert type key |
| enabled | boolean DEFAULT true | Is enabled |
| threshold_value | double precision | Threshold |

**notifications** - Generated alerts

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| company_id | uuid FK | Company |
| device_id | uuid FK | Device |
| alert_type | varchar(32) | Alert type |
| title | varchar(128) | Title |
| message | text | Message |
| severity | varchar(16) | info/warning/critical |
| read | boolean DEFAULT false | Read status |
| data | jsonb | Metadata |
| created_at | timestamptz | Alert time |

**device_events** - Eco scoring, idling, etc.

| Column | Type | Description |
|--------|------|-------------|
| id | bigserial PK | Auto-increment |
| device_id | uuid FK | Device |
| event_type | varchar(32) | Event type |
| title | varchar(128) | Title |
| message | text | Message |
| severity | varchar(16) | Severity |
| latitude, longitude | double precision | Location |
| data | jsonb | Metadata |
| created_at | timestamptz | Event time |

**webhooks** - External registrations

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Auto-generated |
| url | text | Webhook URL |
| secret | text | HMAC secret |
| enabled | boolean DEFAULT true | Is enabled |
| event_types | text | Comma-separated |
| created_at | timestamptz | Creation time |

---

## 8. Deployment & Infrastructure

### 8.1 VPS Deployment

| Property | Value |
|----------|-------|
| Host | `<your-domain>` (configured for your hosting) |
| API | `https://<your-domain>:8081` |
| Frontend | `https://<your-domain>` (or via nginx reverse proxy) |
| Seq | `https://<your-domain>:8083` (internal; not exposed publicly in production) |
| TCP Ingestion | `<your-domain>:5027` (Teltonika), `:5011` (Queclink), `:5023` (GT06) |

### 8.2 Deployment Commands

```bash
# Pull and restart all services
docker-compose pull
docker-compose up -d

# View logs
docker-compose logs -f api
docker-compose logs -f web

# Rebuild after code changes (on build machine)
set DOCKER_BUILDKIT=0
docker-compose build api web
docker-compose up -d
```

### 8.3 Environment Variables

| Variable | Description |
|----------|-------------|
| POSTGRES_PASSWORD | Database password |
| JWT_KEY | JWT signing key (min 32 chars) |
| ConnectionStrings__DefaultConnection | PostgreSQL connection string |
| ASPNETCORE_ENVIRONMENT | Production/Development |
| SEQ_FIRSTRUN_NOAUTHENTICATION | Seq auth (true for dev) |

### 8.4 Mobile Build

```bash
# Capacitor sync (after frontend build)
cd web
npm run build
npx cap sync android

# Build APK
cd android
./gradlew assembleDebug

# Environment for mobile builds
$env:VITE_API_BASE_URL = "https://<your-domain>:8081"
$env:JAVA_HOME = "C:\Program Files\Java\jdk-24"
```

### 8.5 GPS Simulator Build

```bash
cd gps-simulator
./gradlew assembleDebug
# APK at app/build/outputs/apk/debug/app-debug.apk
```

---

## 9. Security

### 9.1 Authentication

- JWT Bearer tokens with configurable signing key
- Passwords hashed via ASP.NET Core Identity (PBKDF2)
- Token contains: user ID, email, roles, company ID
- Tokens validated on every REST and SignalR request

### 9.2 Authorization

- Role-based access control (RBAC): SystemAdmin, Admin, User, Driver
- Company-scoped data isolation on all queries
- Device access layer (DeviceAccess.cs) computes visible devices per role
- Admin endpoints require [Authorize(Roles = "Admin")] or SystemAdmin
- Geofence management restricted to Admin role

### 9.3 Data Isolation

- All queries filtered by company_id for Admin/User/Driver roles
- SystemAdmin bypasses company filter (sees all)
- User/Driver sees only devices linked via user_vehicles table
- company_devices junction table controls multi-company device visibility

### 9.4 Transport Security

- JWT transmitted via HTTP Authorization header
- SignalR uses WebSocket with access_token query parameter
- Production should use HTTPS (TLS termination at reverse proxy)
- Webhook HMAC signing for outbound event verification

### 9.5 Mobile Security

- Biometric authentication (fingerprint/face) via Capacitor plugin
- Credentials encrypted in SecureStorage (Android Keystore)
- Auto-clear credentials on logout
- Biometric check on app launch

---

## 10. Non-Functional Requirements

### 10.1 Performance

| Metric | Target |
|--------|--------|
| Telemetry ingestion latency | < 100ms (device to DB) |
| SignalR push latency | < 200ms (device to client) |
| API response time | < 500ms (95th percentile) |
| Concurrent device connections | 100+ simultaneous TCP connections |
| Database query time | < 50ms for indexed queries |

### 10.2 Scalability

- Horizontal: API containers can be load-balanced (stateless JWT)
- Connection registry is in-memory (single-instance for TCP commands)
- PostgreSQL handles spatial queries with PostGIS GiST indexes
- SignalR can use Redis backplane for multi-instance (not yet configured)

### 10.3 Reliability

- Docker health checks on PostgreSQL (pg_isready)
- Automatic container restart (restart: unless-stopped)
- TCP reconnection in GPS simulator (3s retry on send error)
- Device reconnection handling in DeviceTcpListener
- SignalR auto-reconnect in frontend client

### 10.4 Data Retention

- Telemetry: stored indefinitely (partitioning recommended for production)
- Notifications: stored indefinitely
- Device events: stored indefinitely
- Seq logs: retention configured in Seq UI

### 10.5 Monitoring

- Seq centralized logging (all structured logs)
- Device online/offline detection (configurable threshold)
- Notification system for system alerts
- Power monitoring for device health

---

## 11. Budget & Cost Estimate

### 11.1 Infrastructure Costs (Monthly)

| Service | Spec | Monthly Cost |
|---------|------|-------------|
| VPS | 2 vCPU, 4GB RAM, 80GB SSD | $20-40 |
| PostgreSQL | Included (Docker) | $0 |
| Seq | Included (Docker) | $0 |
| Docker Hosting | Same VPS | $0 |
| Domain + DNS | TLD of your choice | $10-15 |
| SSL Certificate | Let's Encrypt | $0 |
| **Total Monthly** | | **$30-55** |

### 11.2 Hardware Costs (Per Device)

| Item | Unit Cost | Quantity | Total |
|------|-----------|----------|-------|
| Teltonika FMB120 | $50-80 | 10 | $500-800 |
| Teltonika FMB920 | $40-60 | 10 | $400-600 |
| SIM Card (data) | $5-10/month | 20 | $100-200/month |
| OBDII Cable | $10-20 | 20 | $200-400 |
| **Total Hardware (20 devices)** | | | **$1,100-1,800 + $100-200/mo SIM** |

### 11.3 Total Project Cost Summary

| Category | One-Time | Monthly |
|----------|----------|---------|
| Development | TBD | - |
| Infrastructure | - | $30-55 |
| Hardware (20 devices) | $1,100-1,800 | $100-200 (SIM) |
| **Total Year 1** | **TBD** | **$130-255** |
| **Total Year 2+** | - | **$130-255** |

### 11.4 Comparison with Commercial Solutions

| Solution | Setup | Monthly (50 vehicles) |
|----------|-------|----------------------|
| GPSTracker (this) | TBD | $130-255 |
| Teltonika Flespi | $0 | $150-300 |
| GPS-Server.net | $0 | $200-400 |
| Wialon | $0 | $300-600 |
| Gurtam | $0 | $400-800 |

---

## 12. Implementation Roadmap

### Phase 1: Core Platform (Completed)

- [x] Backend API with CQRS (MediatR + Dapper)
- [x] PostgreSQL + PostGIS spatial database
- [x] JWT authentication with role system
- [x] Multi-tenant company structure
- [x] Device auto-registration via TCP
- [x] Teltonika Codec 8/12 protocol support
- [x] Live map with SignalR real-time updates
- [x] Web dashboard (React + TypeScript + TailwindCSS)

### Phase 2: Fleet Management (Completed)

- [x] Vehicle management (CRUD)
- [x] Device-vehicle assignment
- [x] User-vehicle assignment (for User/Driver roles)
- [x] Multi-company device assignment (company_devices)
- [x] Geofencing (circle + polygon)
- [x] Geofence enter/exit detection
- [x] Violation of geofencing tracking

### Phase 3: Intelligence (Partial-completed)

- [x] Notification engine with configurable thresholds
- [~] Speeding, hard brake, hard curve, low battery alerts
- [x] Per-geofence speed limit enforcement
- [ ] Eco scoring (harsh acceleration detection)
- [x] Idling detection (5+ minutes)
- [x] Jamming detection (GNSS status = 0) — as per FMB120 device
- [x] Power monitoring and voltage history
- [x] Digital I/O monitoring (DIN 1-4, DOUT 1-3)

### Phase 4: Remote Control (Completed)

- [x] Codec 12 command builder
- [x] Device connection registry
- [~] Remote command execution (arbitrary) — completed, test failed
- [~] CPU reset (with confirmation) — test failed
- [~] Quick command buttons — test failed
- [~] setparam APN configuration — test failed

### Phase 5: Mobile & Integration (Completed)

- [x] Capacitor Android hybrid app
- [x] Biometric authentication (fingerprint)
- [x] Secure credential storage
- [x] GPS simulator app (standalone Android)
- [x] Webhook system for external integrations
- [x] Seq/Serilog centralized logging

### Phase 6: Future Enhancements (Planned)

- [ ] Fuel consumption estimation
- [ ] Driver behavior scoring/ranking — AI/ML integration
- [ ] Route optimization
- [ ] Scheduled maintenance alerts
- [ ] Multi-language support (i18n)
- [ ] iOS mobile app
- [ ] Push notifications (FCM/APNs)
- [ ] Data export (CSV/Excel/PDF)
- [ ] REST API rate limiting
- [ ] Two-factor authentication (TOTP)
- [ ] Audit logging
- [ ] Dashboard analytics charts
- [ ] Geofence heatmaps
- [ ] Satellite constellation display
- [ ] OBDII parameter extensions
- [ ] Email/SMS notifications
- [ ] Internal client management system (CRM)

---

## 13. Operational Workflow

This section describes the complete onboarding and operational workflow for all roles in the GPS Tracker system.

### 13.1 System Bootstrap (Automatic)

The system auto-seeds a **SystemAdmin** account and a **Default** company on first API startup.

- **Credentials**: `admin@gpstracker.local` / `P@ssw0rd` (from `appsettings.json`)
- **Default company**: "Default" is auto-inserted into the `companies` table

### 13.2 SystemAdmin Setup

| Step | Action | Description |
|------|--------|-------------|
| 1.1 | **Login** | Sign in as `admin@gpstracker.local`. You land on the Live Map — all devices across all companies are visible. |
| 1.2 | **Create companies** | Go to `Administration → Companies`. Create companies for each customer/organization. Each company gets its own isolated fleet of vehicles, devices, users, and geofences. |
| 1.3 | **Tune company thresholds** | Still on Companies, adjust `offlineThresholdMinutes` (when to flag "no signal") and `tripIgnitionGapMinutes` (trip detection sensitivity) per company. These affect all devices and users under that company. |
| 1.4 | **Assign an Admin to each company** | Go to `Administration → Companies → Users` for a company. Click "Add" next to an available user to assign them. This person will manage that company's fleet. |

> **Note**: Users must self-register first before a SystemAdmin can assign them.

### 13.3 User Registration (Self-Service)

| Step | Action | Description |
|------|--------|-------------|
| 2.1 | **Register** | Any person navigates to the app and clicks "Register." They provide email + password + display name. The account is created with a `User` role and no company — shown as "Pending" in the admin panel. |
| 2.2 | **Login** | After registration, the user is auto-logged in. Without a company assignment, they see an empty map and no devices. They wait to be linked to a company. |

### 13.4 Device Onboarding (Teltonika Hardware)

| Step | Action | Description |
|------|--------|-------------|
| 3.1 | **Power on the device** | A Teltonika FMB-120 (or compatible) is installed in a vehicle and powered on. It auto-connects to the TCP server on port 5027. |
| 3.2 | **Auto-registration** | The server receives the IMEI handshake, auto-creates a device record with `display_name = IMEI`, and broadcasts a real-time toast to all SystemAdmins: *"New device registered: {IMEI}"*. No manual registration needed. |
| 3.3 | **Rename (optional)** | SystemAdmin opens `Devices → {device} → Rename` to give it a meaningful name (e.g., "Truck 01"). |
| 3.4 | **Assign device to company** | On the device detail page, SystemAdmin clicks `Companies → Edit` and selects the company. The device now appears in that company's fleet. A device can belong to multiple companies simultaneously. |

> **Alternative**: SystemAdmin can also manually create a device from the `Devices` page using the "Create device" button (enters IMEI + display name + protocol).

### 13.5 Company Admin Fleet Setup

Performed by the **Company Admin** assigned in Step 1.4.

| Step | Action | Description |
|------|--------|-------------|
| 4.1 | **Create vehicles** | Go to `Vehicles → Create vehicle`. Give each vehicle a name (e.g., "Van 12"). A vehicle is a logical container — it will be linked to a device and assigned to drivers. |
| 4.2 | **Link device to vehicle** | On the `Vehicles` page, click "Link device" for a vehicle. Select the unlinked device (by IMEI or display name) that is already assigned to the company (Step 3.4). One device can link to only one vehicle at a time. |
| 4.3 | **Link registered users** | Go to `Admin → Link existing`. Enter the email of a user who self-registered. This links them to your company so they can see your fleet. |
| 4.4 | **Set user role** | For each linked user, click "Role" and assign one of: |
| | | • **User** — can view all company vehicles on the map |
| | | • **Admin** — can manage users, devices, vehicles, and geofences for this company |
| | | • **Driver** — sees only their assigned vehicles |
| 4.5 | **Assign vehicles to drivers** | For users with the **Driver** role, click "Vehicles" and check the vehicles they should see. This is what controls their map view — they only see assigned vehicles. |

### 13.6 Ongoing Operations

#### SystemAdmin

| Step | Action | Description |
|------|--------|-------------|
| 5.1 | **Monitor devices** | `Devices` page — filter/search all devices across all companies, check online/offline status, last seen time. |
| 5.2 | **Manage company membership** | `Administration → Companies → Users` — add or remove users from any company. |
| 5.3 | **Activate/deactivate companies** | Toggle a company's active status. Inactive companies are visible to admins but their users may lose access. |
| 5.4 | **Send remote commands** | `Devices → {device} → Remote commands` — send CPU reset, output control, or parameter changes to online Teltonika devices via Codec 12 over TCP. |
| 5.5 | **Delete users/companies** | Remove a user entirely (`Administration → Users → Remove`) or delete a company and all its data (`Administration → Companies → Remove`). |
| 5.5a | **Manage system users** | `Administration → Users` tab — view all registered users, set company assignment, update display names, delete users. |
| 5.5b | **Manage system companies** | `Administration → Companies` tab — create/edit/delete companies, assign company admins, toggle active status. |

#### Company Admin

| Step | Action | Description |
|------|--------|-------------|
| 5.6 | **Manage users** | `Admin` — link new users, change roles, remove users from the company. |
| 5.7 | **Manage vehicles** | `Vehicles` — create, rename, link/unlink devices, remove vehicles. |
| 5.8 | **Manage devices** | `Devices` — rename devices, set phone numbers, adjust offline thresholds. |
| 5.9 | **Configure geofences** | `Geofences` — create geofence zones (circle or polygon), assign vehicles, set speed limits. Violations are logged and trigger notifications. |
| 5.10 | **Set up webhooks** | `Webhooks` — register HTTP endpoints to receive real-time geofence and location events. |
| 5.11 | **Adjust settings** | `Settings` — company-wide offline threshold, trip gap, notification preferences. |

#### User

| Step | Action | Description |
|------|--------|-------------|
| 5.12 | **View live map** | `Map` — see all company vehicles in real-time with position, speed, and status. |
| 5.13 | **Check dashboard** | `Dashboard` — overview of vehicle count, online/offline stats, recent geofence events. |
| 5.14 | **View device details** | `Devices → {device}` — position history map, trip list, power log, device events, geofence violations, live telemetry status. |

#### Driver

| Step | Action | Description |
|------|--------|-------------|
| 5.15 | **View assigned vehicles** | `Map` — only their assigned vehicles appear (filtered by `user_vehicles` table). |
| 5.16 | **View vehicle details** | `Devices → {device}` — same as User but scoped to assigned vehicles only (history, trips, events, power log, violations). |

### 13.7 Summary Flow

```
SystemAdmin                          Hardware              Company Admin
    |                                    |                      |
    +-- 1. Create company                |                      |
    +-- 2. User self-registers           |                      |
    +-- 3. Assign user to company        |                      |
    |                                    |                      |
    |              4. Device powers on -->                      |
    |              5. Auto-registers                           |
    +-- 6. Rename device (opt.)          |                      |
    +-- 7. Assign device --> company     |                      |
    |                                    |                      |
    |                                    +-- 8. Create vehicle  |
    |                                    +-- 9. Link device     |
    |                                    +-- 10. Link users     |
    |                                    +-- 11. Set roles      |
    |                                    +-- 12. Assign drivers |
    |                                    |                      |
    |                                    +-- 13. Create geofences
    |                                    +-- 14. Set notifications
    |                                    |                      |
    |         <------- Live map + real-time tracking -----------+
```

### 13.8 Role Summary

| Role | Scope | Map View | Can Manage |
|------|-------|----------|------------|
| **SystemAdmin** | Global (all companies) | All devices | Users, companies, devices |
| **Admin** | Company-scoped | All company devices | Users, vehicles, devices, geofences, webhooks, settings |
| **User** | Company-scoped | All company devices | Nothing (read-only) |
| **Driver** | Company-scoped | Assigned vehicles only | Nothing (read-only) |

### 13.9 Key Technical Details

- **Device visibility** is controlled by `DeviceAccess.cs` — a single source of truth used by both REST API and SignalR real-time push.
- **Teltonika protocol**: TCP port 5027, IMEI handshake (2-byte BE length + ASCII IMEI, server responds `0x01`), Codec 8 frames.
- **Multi-company devices**: A single device can be assigned to multiple companies via the `company_devices` junction table.
- **One device per vehicle**: The `ux_vehicles_device` unique index enforces that a device links to at most one vehicle at a time.
- **Role reassignment**: Setting a user's role replaces ALL their current roles with the single new one. `SystemAdmin` can only be granted via bootstrap seeding.

