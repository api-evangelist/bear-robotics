---
name: Receive robot events by webhook
description: Register, filter, shape and operate a Bear Cloud API webhook subscription so an HTTP-only system is pushed mission and battery events instead of holding a gRPC stream.
api: grpc/v1/bear-robotics-services-cloud-api_service.proto
operations: [CreateWebhook, ListWebhooks, DeleteWebhook]
generated: '2026-08-06'
method: generated
source: https://cloud.api.bearrobotics.ai/v1.3/resources/Webhooks/
---

# Receive robot events by webhook

New in API v1.3. The HTTP-native alternative to a long-lived gRPC stream — but it covers only
**two** event types.

> **Before you start.** The three webhook operations are **not in Bear's published OpenAPI
> document** and **not in its Postman collection**. They exist in the published proto
> (`bearrobotics.api.v1.services.cloud.APIService`) and the REST paths are given in docs prose
> only. There is no machine-readable schema for these three calls — the field lists below come
> from the proto and the documentation, and you should expect to hand-write the request bodies.

## 1. Stand up a receiver first

Requirements the API enforces at creation time:

- **HTTPS only**, and the host must resolve to a publicly reachable address. Loopback, private
  and link-local addresses are rejected.
- Respond **2xx quickly**. Anything else counts as a failure, and 20 consecutive final failures
  auto-disable the subscription.
- **Deliveries are not signed.** There is no HMAC and no timestamp header, so there is no
  replay window to verify. The documented pattern is a custom header carrying a shared secret
  or bearer token, checked together with the request arriving over HTTPS. Configure it via
  `options.headers`.
- **Deduplicate on `X-Bear-Webhook-Event-Id`.** It is stable across retries of the same event.
  This is the only idempotency the system offers.

## 2. Create the subscription

`CreateWebhook` (`POST /v1/webhook/create`):

```json
{
  "url": "https://example.com/bear/webhooks",
  "eventType": "mission",
  "selector": { "robotIds": { "ids": ["pennybot-abc123"] } },
  "filter": {
    "field": "state.current_mission.state",
    "operator": "FILTER_OPERATOR_IN",
    "values": ["STATE_SUCCEEDED", "STATE_FAILED", "STATE_CANCELED"]
  },
  "options": {
    "description": "Notify on mission completion",
    "headers": { "x-secret": "<your shared secret>" }
  }
}
```

- `event_type` is `"mission"` or `"battery"` — those are the only two.
- `selector` is a oneof: exactly one of `robot_ids` or `location_ids`. Both, or neither,
  returns `INVALID_ARGUMENT`.
- `filter` holds **one** condition. There is no AND/OR — create separate subscriptions.
  `FILTER_OPERATOR_IN` is the only operator. `robot_id` is not filterable; scope robots with
  the selector instead.

## 3. Mind the casing — this is the trap

The management API request and response use **camelCase** (`robotId`, `eventType`). The
**delivered body uses snake_case** proto names (`robot_id`, `current_mission_index`,
`battery_state`). Filter paths and template placeholders address the snake_case body. Build the
receiver against snake_case.

Default delivered body:

```json
{ "robot_id": "pennybot-abc123", "state": { } }
```

Scalars are always present, enums render by name (`"STATE_SUCCEEDED"`), and unset nested
messages arrive as `null` — a filter path into a `null` message resolves empty and never
matches an `IN`.

## 4. Reshape the body if you need to

`options.request_template` is any JSON object; `{{field_path}}` placeholders resolve to
`robot_id` or a **scalar** under `state.*`, may sit inside surrounding text, and may be nested.
Pointing a placeholder at a whole message, list or map is rejected at creation. Index into a
list to reach a scalar: `{{state.missions.0.mission_id}}`. Missing paths render as `""`.
Placeholders are **never** substituted in header values.

## 5. Operate it

- `ListWebhooks` (`POST /v1/webhook/list`) — all subscriptions for the distributor.
- `DeleteWebhook` (`POST /v1/webhook/delete`) — soft delete by `id`.
- There is **no update and no re-enable call.** Poll `ListWebhooks` for `enabled: false` with a
  `disabledReason`; to recover, fix the endpoint, delete the subscription and create a new one.

Retry classification: network errors, timeouts, `408`, `425`, `429` and any `5xx` are retried
with exponential backoff. Every other non-2xx — including most `4xx` and `3xx` redirects, which
are not followed — is permanent and is not retried.

## 6. Know what you are not getting

Ten of the twelve streaming signals have no webhook equivalent — e-stop, error codes,
navigation-stuck, online status, localization, network, pose, tray and conveyor state. For
those, hold a gRPC stream or poll `APIService_GetRobotStatus`.
