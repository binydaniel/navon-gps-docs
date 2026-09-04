# GPS Tracker — Full Workflow Guide

This document describes the complete onboarding and operational workflow for all roles in the GPS Tracker system.

- [Home](README.md)
- [Software Requirements Specification (SRS)](SRS.md)

---

## Phase 0: System Bootstrap (Automatic)

The system auto-seeds a **SystemAdmin** account and a **Default** company on first API startup.

- **Credentials**: `admin@gpstracker.local` / `P@ssw0rd` (from `appsettings.json`)
- **Default company**: "Default" is auto-inserted into the `companies` table

---

## Phase 1: SystemAdmin Setup

| Step | Action | Description |
|------|--------|-------------|
| 1.1 | **Login** | Sign in as `admin@gpstracker.local`. You land on the Live Map — all devices across all companies are visible. |
| 1.2 | **Create companies** | Go to `Administration → Companies`. Create companies for each customer/organization. Each company gets its own isolated fleet of vehicles, devices, users, and geofences. |
| 1.3 | **Tune company thresholds** | Adjust `offlineThresholdMinutes` (when to flag "no signal") and `tripIgnitionGapMinutes` (trip detection sensitivity) per company. These affect all devices and users under that company. |
| 1.4 | **Assign an Admin to each company** | Go to `Administration → Companies → Users` for a company. Click "Add" next to an available user to assign them. |

> **Note**: Users must self-register first (Phase 2) before a SystemAdmin can assign them.

---

## Phase 2: User Registration (Self-Service)

| Step | Action | Description |
|------|--------|-------------|
| 2.1 | **Register** | Any person navigates to the app and clicks "Register." They provide email + password + display name. The account is created with a `User` role and no company — shown as "Pending" in the admin panel. |
| 2.2 | **Login** | After registration, the user is auto-logged in. Without a company assignment, they see an empty map and no devices. They wait to be linked. |

---

## Phase 3: Device Onboarding (Teltonika Hardware)

| Step | Action | Description |
|------|--------|-------------|
| 3.1 | **Power on the device** | A Teltonika FMB-120 (or compatible) is installed in a vehicle and powered on. It auto-connects to the TCP server on port 5027. |
| 3.2 | **Auto-registration** | The server receives the IMEI handshake, auto-creates a device record with `display_name = IMEI`, and broadcasts a real-time toast: *"New device registered: {IMEI}"*. No manual registration needed. |
| 3.3 | **Rename (optional)** | SystemAdmin opens `Devices → {device} → Rename` to give it a meaningful name (e.g., "Truck 01"). |
| 3.4 | **Assign device to company** | On the device detail page, SystemAdmin clicks `Companies → Edit` and selects the company. A device can belong to multiple companies simultaneously. |

> **Alternative**: SystemAdmin can also manually create a device from the `Devices` page using the "Create device" button.

---

## Phase 4: Company Admin Fleet Setup

Performed by the **Company Admin** assigned in Step 1.4.

| Step | Action | Description |
|------|--------|-------------|
| 4.1 | **Create vehicles** | Go to `Vehicles → Create vehicle`. Give each vehicle a name. A vehicle is a logical container linked to a device and assigned to drivers. |
| 4.2 | **Link device to vehicle** | On the `Vehicles` page, click "Link device". Select an unlinked device already assigned to the company. One device links to only one vehicle at a time. |
| 4.3 | **Link registered users** | Go to `Admin → Link existing`. Enter the email of a user who self-registered. |
| 4.4 | **Set user role** | For each linked user, click "Role" and assign one of: |
| | | • **User** — can view all company vehicles on the map |
| | | • **Admin** — can manage users, devices, vehicles, and geofences for this company |
| | | • **Driver** — sees only their assigned vehicles |
| 4.5 | **Assign vehicles to drivers** | For users with the **Driver** role, click "Vehicles" and check the vehicles they should see. |

---

## Phase 5: Ongoing Operations

### SystemAdmin

| Step | Action | Description |
|------|--------|-------------|
| 5.1 | **Monitor devices** | `Devices` page — filter/search all devices across all companies, check online/offline status, last seen time. |
| 5.2 | **Manage company membership** | `Administration → Companies → Users` — add or remove users from any company. |
| 5.3 | **Activate/deactivate companies** | Toggle a company's active status. |
| 5.4 | **Send remote commands** | `Devices → {device} → Remote commands` — send CPU reset, output control, or parameter changes to online Teltonika devices via Codec 12 over TCP. |
| 5.5 | **Delete users/companies** | Remove a user entirely (`Administration → Users → Remove`) or delete a company and all its data (`Administration → Companies → Remove`). |
| 5.5a | **Manage system users** | `Administration → Users` tab — view all registered users, set company assignment, update display names, delete users. |
| 5.5b | **Manage system companies** | `Administration → Companies` tab — create/edit/delete companies, assign company admins, toggle active status. |

### Company Admin

| Step | Action | Description |
|------|--------|-------------|
| 5.6 | **Manage users** | `Admin` — link new users, change roles, remove users from the company. |
| 5.7 | **Manage vehicles** | `Vehicles` — create, rename, link/unlink devices, remove vehicles. |
| 5.8 | **Manage devices** | `Devices` — rename devices, set phone numbers, adjust offline thresholds. |
| 5.9 | **Configure geofences** | `Geofences` — create geofence zones (circle or polygon), assign vehicles, set speed limits. Violations trigger notifications. |
| 5.10 | **Adjust settings** | `Settings` — company-wide offline threshold, trip gap, notification preferences. |

### User

| Step | Action | Description |
|------|--------|-------------|
| 5.11 | **View live map** | `Map` — see all company vehicles in real-time with position, speed, and status. |
| 5.12 | **Check dashboard** | `Dashboard` — overview of vehicle count, online/offline stats, recent geofence events. |
| 5.13 | **View device details** | `Devices → {device}` — position history map, trip list, power log, device events, geofence violations, live telemetry. |

### Driver

| Step | Action | Description |
|------|--------|-------------|
| 5.14 | **View assigned vehicles** | `Map` — only their assigned vehicles appear (filtered by `user_vehicles` table). |
| 5.15 | **View vehicle details** | `Devices → {device}` — same as User but scoped to assigned vehicles only. |

---

## Summary Flow

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

---

## Role Summary

| Role | Scope | Map View | Can Manage |
|------|-------|----------|------------|
| **SystemAdmin** | Global (all companies) | All devices | Users, companies, devices |
| **Admin** | Company-scoped | All company devices | Users, vehicles, devices, geofences, settings |
| **User** | Company-scoped | All company devices | Nothing (read-only) |
| **Driver** | Company-scoped | Assigned vehicles only | Nothing (read-only) |

---

## Key Technical Details

- **Device visibility** is controlled by `DeviceAccess.cs` — a single source of truth used by both REST API and SignalR real-time push.
- **Teltonika protocol**: TCP port 5027, IMEI handshake (2-byte BE length + ASCII IMEI, server responds `0x01`), Codec 8 frames.
- **Multi-company devices**: A single device can be assigned to multiple companies via the `company_devices` junction table.
- **One device per vehicle**: The `ux_vehicles_device` unique index enforces that a device links to at most one vehicle at a time.
- **Role reassignment**: Setting a user's role replaces ALL their current roles with the single new one. `SystemAdmin` can only be granted via bootstrap seeding.