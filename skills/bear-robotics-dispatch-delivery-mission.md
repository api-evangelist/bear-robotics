---
name: Dispatch a Servi delivery mission
description: Authenticate against the Bear Cloud API, find a robot at a location, confirm it is fit to move, and enqueue a Servi delivery mission to a named destination.
api: openapi/bear-robotics-cloud-openapi-original.yml
operations: [APIService_GetAvailableLocations, APIService_ListRobotIDs, APIService_GetRobotStatus, APIService_GetCurrentMap, APIService_CreateMission]
generated: '2026-08-06'
method: generated
source: openapi/bear-robotics-cloud-openapi-original.yml + https://cloud.api.bearrobotics.ai/
---

# Dispatch a Servi delivery mission

Moves a physical robot through a space that contains people. Treat every step as
consequential and never retry a write blindly — see **Safety** below.

## 1. Get a token

`POST https://api-auth.bearrobotics.ai/authorizeApiAccess` with the credentials JSON
(`api_key`, `secret`, `scope`). The response is a JWT. Send it on every call as
`Authorization: Bearer <JWT>`. Refresh it roughly every 30 minutes — it carries an `exp`.

API keys are not self-serve; they are issued by a Bear Robotics account manager.

## 2. Find the site

`APIService_GetAvailableLocations` (`POST /v1/available-locations/get`) returns the locations
your distributor can reach. Pick a `location_id`.

`APIService_GetLocationInfo` (`POST /v1/location-info/get`) expands it into floors, sections
and the `current_map_id` for each section.

## 3. Find a robot

`APIService_ListRobotIDs` (`POST /v1/robot-ids/list`) with a `RobotFilter` carrying the
`location_id`. This is the only filter available and the list is unpaged.

## 4. Confirm the robot can take the job — do not skip this

`APIService_GetRobotStatus` (`POST /v1/robot-state/get`) returns the consolidated `RobotState`.
Before enqueuing, check:

- `robot_connection` — an offline robot will accept nothing.
- `emergency_stop_state` — a robot under e-stop must not be dispatched.
- `battery_state.charge_percent` — if it is low, run
  `APIService_ChargeRobot` (`POST /v1/robot/charge`) instead of a delivery.
- `mission_state` — the robot may already have a queue. Use
  `APIService_AppendMission` rather than `APIService_CreateMission` if you intend to add to it.
- `navigation_state` (v1.3) — carries the robot's stuck state.
- `localization_state` — a robot that has lost localization will not navigate.

## 5. Resolve the destination

`APIService_GetCurrentMap` (`POST /v1/current-map/get`) for the robot returns the `Map`, whose
`annotations` carry the `destinations`. Use a real `destination_id` from that map — never
invent one, and never assume a destination on one map exists on another.

## 6. Create the mission

`APIService_CreateMission` (`POST /v1/mission/create`) with `robot_id` and a `Mission` body.

`Mission` is a three-level discriminated union and this is where most integrations go wrong:

- pick the product family — `servi_mission` for Servi, `carti_mission` for Carti,
  `base_mission` for a plain navigate;
- inside `ServiMission`, pick the kind — `delivery_mission`, `bussing_mission`,
  `delivery_patrol_mission`, `bussing_patrol_mission`, `navigate_mission`,
  `navigate_auto_mission`;
- `DeliveryMission` carries `DeliveryParams`, whose `tray_mappings` bind each tray to a `Goal`,
  and each `Goal` carries the destination.

The response returns the server-assigned `mission_id`. To enqueue several stops atomically use
`APIService_CreateMissionBatch` (`POST /v1/mission/create-batch`) — all-or-nothing within the
one call.

## 7. Watch it

There is no REST way to subscribe. Choose one:

- **gRPC** `SubscribeMissionStatus` on `api.bearrobotics.ai:443` — event-based, closes after
  60 minutes, so implement a conditional retry.
- **Webhooks** (v1.3) — `CreateWebhook` (`POST /v1/webhook/create`, gRPC-only in the spec) with
  `event_type: "mission"` and a filter on `state.current_mission.state` for
  `["STATE_SUCCEEDED","STATE_FAILED","STATE_CANCELED"]`. See
  `bear-robotics-receive-robot-events.md`.
- **Polling** `APIService_GetRobotStatus`.

To abandon the current stop, `APIService_SkipGoal` (`POST /v1/goal/skip`).

## Safety

- **There is no idempotency key.** A command RPC that times out returns `DEADLINE_EXCEEDED`
  (HTTP 504) after 10 seconds, and the timeout fires at the cloud service — not at the robot.
  The mission may already be queued. Before retrying `APIService_CreateMission`, re-read
  `APIService_GetRobotStatus` and inspect the queue.
- **There is no sandbox.** Every call reaches a real robot.
- Errors are gRPC canonical codes: `UNAUTHENTICATED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`,
  `FAILED_PRECONDITION` (robot firmware too old for this API version — v1.3 needs servi-26.02),
  `DEADLINE_EXCEEDED`, `INTERNAL`. See `errors/bear-robotics-problem-types.yml`.
