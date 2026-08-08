---
name: Monitor fleet health
description: Read the live state of every robot at a location — connection, battery, e-stop, localization, navigation-stuck and error codes — using the Bear Cloud API's read-only operations.
api: openapi/bear-robotics-cloud-openapi-original.yml
operations: [APIService_GetAvailableLocations, APIService_ListRobotIDs, APIService_GetRobotStatus, APIService_GetRobotSystemInfo, APIService_GetLocationInfo]
generated: '2026-08-06'
method: generated
source: openapi/bear-robotics-cloud-openapi-original.yml + https://cloud.api.bearrobotics.ai/
---

# Monitor fleet health

Entirely read-only. Safe for an autonomous agent.

## 1. Token

`POST https://api-auth.bearrobotics.ai/authorizeApiAccess` with `api_key` / `secret` / `scope`;
send the JWT as `Authorization: Bearer <JWT>` and refresh every ~30 minutes.

## 2. Enumerate

- `APIService_GetAvailableLocations` (`POST /v1/available-locations/get`) — the sites you can see.
- `APIService_ListRobotIDs` (`POST /v1/robot-ids/list`) — robots, optionally filtered by
  `location_id` via `RobotFilter`. Unpaged.
- `APIService_GetLocationInfo` (`POST /v1/location-info/get`) — floors, sections, current map ids.

## 3. Read state per robot

`APIService_GetRobotStatus` (`POST /v1/robot-state/get`) returns `RobotState`:

| Field | What it tells you |
|---|---|
| `robot_connection` | whether the cloud can currently reach the robot |
| `battery_state` | `charge_percent`, `state`, `charge_method` |
| `emergency_stop_state` | whether the robot is halted by e-stop |
| `mission_state` | the active mission and its current goal |
| `pose` / `twist` | where it is and how it is moving |
| `localization_state` | whether the robot knows where it is (v1.3) |
| `navigation_state` | whether the robot is stuck, and why when known (v1.3) |
| `error_codes` | active fault codes |
| `servi_state` | tray states, on Servi |
| `carti_state` | conveyor state, on Carti |

`APIService_GetRobotSystemInfo` (`POST /v0/robot-system-info/get`) returns the static
`SystemInfo`, including the software version — worth caching, and worth checking when a call
fails with `FAILED_PRECONDITION` (v1.3 requires servi-26.02 / carti-26.02).

## 4. Prefer streams over polling

Polling `GetRobotStatus` per robot does not scale. If you can hold a gRPC connection, use the
server-streaming RPCs on `api.bearrobotics.ai:443`:

- `SubscribeRobotStatus` — consolidated, ~1Hz, best for a dashboard.
- `SubscribeOnlineStatus` / `SubscribeNavigationStatus` — fleet-wide, selected by robot ids or
  location (v1.3).
- `SubscribeRobotPose` (~10Hz), `SubscribeBatteryStatus`, `SubscribeErrorCodes`,
  `SubscribeEmergencyStopStatus`, `SubscribeLocalizationStatus`, `SubscribeNetworkStatus`,
  `SubscribeTrayStatuses`, `SubscribeConveyorStatus` — narrower and higher-frequency.

Streams close after 60 minutes; implement a conditional retry. Delivery is **best effort, not
at-least-once** — updates can be dropped. Every response carries `EventMetadata` with a
`timestamp` and a `sequence_number`; the sequence number detects duplicates but may reset to 0
on a restart, and the timestamp is the robot's local clock, so it must not be used to order
events across robots.

## 5. What you cannot get over HTTP

Only `mission` and `battery` are available as webhooks. An HTTP-only integration cannot be
pushed e-stop, error codes, navigation-stuck, tray or conveyor events — it must poll
`GetRobotStatus` for those.
