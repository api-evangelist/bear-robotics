---
name: Manage maps and locations
description: Walk a Bear Robotics site from location to floor to section to map, read a map's annotations and destinations, and switch or re-localize a robot onto a map.
api: openapi/bear-robotics-cloud-openapi-original.yml
operations: [APIService_GetAvailableLocations, APIService_GetLocationInfo, APIService_GetCurrentMap, APIService_GetMap, APIService_SwitchMap, APIService_LocalizeRobot, APIService_SetPose]
generated: '2026-08-06'
method: generated
source: openapi/bear-robotics-cloud-openapi-original.yml + https://cloud.api.bearrobotics.ai/concepts/location/
---

# Manage maps and locations

Reads are safe. The three writes at the end change how a robot understands where it is, and a
wrong one sends it to the wrong place.

## 1. Token

`POST https://api-auth.bearrobotics.ai/authorizeApiAccess`; send `Authorization: Bearer <JWT>`.

## 2. Walk the hierarchy

`Location → floors → sections → current map`.

- `APIService_GetAvailableLocations` (`POST /v1/available-locations/get`) — the `location_id`s
  your distributor can reach.
- `APIService_GetLocationInfo` (`POST /v1/location-info/get`) — returns `Location` with
  `Location_Floor`s, each carrying `Floor_Section`s, each carrying `current_map_id`.

## 3. Read a map

- `APIService_GetCurrentMap` (`POST /v1/current-map/get`) — by `robot_id`, the map that robot is
  navigating on right now.
- `APIService_GetMap` (`POST /v1/map/get`) — by `map_id`.

A `Map` carries `annotations` (each with `destinations`, and each `Destination` with a
`destination_id` and a `Pose`), an `origin`, and `MapImageDownloadInfo` with a `SignedURL` and
`MapImageFileInfo`.

**Verify the image.** From v1.3 `MapImageFileInfo` carries `md5_checksum`, a 32-character
lowercase hex string. The older CRC32C `checksum` field is deprecated — read `md5_checksum`.

Destination ids are scoped to their map. A `destination_id` valid on one map is not valid on
another; always resolve destinations from the map the robot is actually on.

## 4. Writes — handle with care

- `APIService_SwitchMap` (`POST /v1/map/switch`) — moves a robot onto a different map. It
  returns the new `map_id`. Every queued goal referencing the old map's destinations becomes
  meaningless; drain or clear the mission queue first.
- `APIService_LocalizeRobot` (`POST /v1/robot/localize`) — re-localizes the robot to a `Goal`.
  Use it when `localization_state` on `RobotState` shows the robot has lost its position.
- `APIService_SetPose` (`POST /v1/pose/set`) — sets the robot's estimated pose directly. A wrong
  pose makes the robot confidently navigate from a position it is not in. Read the current pose
  from `APIService_GetRobotStatus` first, and prefer `LocalizeRobot` where it will do.

## 5. After any of the three

Re-read `APIService_GetRobotStatus` and confirm `localization_state` and `pose` before
enqueuing work. None of these calls is idempotent and none can be undone by replaying it.
