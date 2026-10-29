# ArmLink Controller API Reference

> **Portfolio note:** This is an illustrative documentation sample. ArmLink is a designed REST API for a 6-DOF robotic arm controller, used to demonstrate API reference writing, hardware fault documentation, and evaluation of AI-generated documentation. Fault behaviours reflect common industrial servo-drive practice.

The ArmLink API lets client applications read joint state, command motion, and monitor hardware faults on a 6-axis robotic arm controller over HTTP.

## Contents

1. [API Endpoint Reference](#1-api-endpoint-reference)
2. [Real-Time Failure & Error State Specification](#2-real-time-failure--error-state-specification)
3. [Sample Evaluation: Two Candidate Documentation Sets](#3-sample-evaluation-two-candidate-documentation-sets)

---

## 1. API Endpoint Reference

### Basics

| Item | Value |
|---|---|
| Base URL | `http://<controller-ip>:8080/api/v1` |
| Format | JSON (`Content-Type: application/json`) |
| Authentication | Bearer token: `Authorization: Bearer <token>` |
| Angles | Degrees |
| Velocities | Degrees per second (joint) or millimetres per second (Cartesian) |
| Positions | Millimetres, in the robot base frame |
| Temperatures | Degrees Celsius |
| Currents | Amperes |
| Timestamps | ISO 8601, UTC, millisecond precision |
| Joint IDs | Integers `1`–`6`, base to wrist |

### Joint limits

| Joint | Name | Range (°) | Max velocity (°/s) |
|---|---|---|---|
| 1 | Base | −170 to +170 | 180 |
| 2 | Shoulder | −120 to +120 | 150 |
| 3 | Elbow | −150 to +150 | 180 |
| 4 | Wrist roll | −180 to +180 | 300 |
| 5 | Wrist pitch | −120 to +120 | 300 |
| 6 | Tool flange | −360 to +360 | 360 |

### Endpoint summary

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/status` | Overall arm state and active fault count |
| `GET` | `/joints` | State of all six joints |
| `GET` | `/joints/{id}` | State of one joint |
| `POST` | `/drives/enable` | Power the motor drives |
| `POST` | `/drives/disable` | Remove drive power |
| `POST` | `/motion/joint` | Move to target joint angles |
| `POST` | `/motion/stop` | Controlled stop of all motion |
| `GET` | `/faults` | List active and recent faults |
| `POST` | `/faults/clear` | Clear latched faults that are safe to clear |

---

### `GET /status`

Returns the arm's current state.

**Response `200 OK`**

```json
{
  "state": "READY",
  "enabled": true,
  "active_faults": 0,
  "velocity_override": 1.0,
  "uptime_s": 18342,
  "timestamp": "2026-10-16T14:02:11.482Z"
}
```

| Field | Type | Description |
|---|---|---|
| `state` | string | One of the states in [Arm state machine](#arm-state-machine). |
| `enabled` | boolean | `true` if the motor drives are powered. |
| `active_faults` | integer | Number of faults currently active. |
| `velocity_override` | number | Speed scale applied to all motion, `0.0`–`1.0`. Drops below `1.0` when the controller derates for thermal protection. |
| `uptime_s` | integer | Seconds since controller boot. |
| `timestamp` | string | Time the status was sampled. |

---

### `GET /joints`

Returns the state of all six joints, in order 1–6.

**Query parameters:** `include` (`basic` or `full`), as for [`GET /joints/{id}`](#get-jointsid).

**Response `200 OK`** (`?include=basic`)

```json
{
  "joints": [
    { "id": 1, "name": "base",         "position_deg": 0.00,  "velocity_dps": 0.0, "status": "OK" },
    { "id": 2, "name": "shoulder",     "position_deg": -45.02, "velocity_dps": 0.0, "status": "OK" },
    { "id": 3, "name": "elbow",        "position_deg": 42.15, "velocity_dps": 0.0, "status": "OK" },
    { "id": 4, "name": "wrist_roll",   "position_deg": 0.00,  "velocity_dps": 0.0, "status": "OK" },
    { "id": 5, "name": "wrist_pitch",  "position_deg": 45.00, "velocity_dps": 0.0, "status": "OK" },
    { "id": 6, "name": "tool_flange",  "position_deg": 0.00,  "velocity_dps": 0.0, "status": "OK" }
  ],
  "timestamp": "2026-10-16T14:02:11.482Z"
}
```

All six joints are sampled in the same control cycle, so their values are consistent with each other. Polling `/joints/{id}` six times does not give that guarantee.

---

### `GET /joints/{id}`

Returns the state of one joint.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | integer | Joint ID, `1`–`6`. |

**Query parameters**

| Name | Type | Default | Description |
|---|---|---|---|
| `include` | string | `basic` | `basic` returns position and velocity. `full` adds temperature, current, and both encoder readings. |

**Response `200 OK`** (`?include=full`)

```json
{
  "id": 3,
  "name": "elbow",
  "position_deg": 42.15,
  "velocity_dps": 0.0,
  "motor_encoder_deg": 42.15,
  "output_encoder_deg": 42.13,
  "temperature_c": 51.2,
  "current_a": 1.84,
  "status": "OK"
}
```

`motor_encoder_deg` is the motor-side encoder reading converted through the gear ratio. `output_encoder_deg` is read directly at the joint output. A growing gap between them indicates slip (see `ENCODER_SLIP`).

**Errors:** `404 JOINT_NOT_FOUND` if `id` is outside `1`–`6`.

---

### `POST /drives/enable`

Powers the motor drives and moves the arm from `DISABLED` to `READY`. This is the only way to enable the drives. The controller never does it on its own, including after faults are cleared.

**Response `200 OK`**

```json
{ "state": "READY", "enabled": true }
```

**Errors**

| Status | Code | When |
|---|---|---|
| `409` | `FAULTS_ACTIVE` | One or more faults are still active. Call `GET /faults` to see them. |
| `409` | `ESTOP_ENGAGED` | The hardware emergency stop is engaged. |

---

### `POST /drives/disable`

Stops any motion with a controlled stop, then removes drive power. The arm moves to `DISABLED`. Always succeeds, and returns `200 OK` with `{ "state": "DISABLED", "enabled": false }`.

---

### `POST /motion/joint`

Moves the arm to target joint angles. The request returns as soon as the motion is accepted. It does not wait for the motion to finish.

**Request body**

```json
{
  "targets_deg": [0.0, -45.0, 90.0, 0.0, 45.0, 0.0],
  "velocity_scale": 0.5,
  "blocking": false
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `targets_deg` | array of 6 numbers | Yes | — | Target angle for each joint, in order 1–6. |
| `velocity_scale` | number | No | `0.25` | Fraction of each joint's max velocity, `0.01`–`1.0`. Multiplied by `velocity_override`. |
| `blocking` | boolean | No | `false` | If `true`, the response is sent only after the motion completes or fails. |

**Response `202 Accepted`**

```json
{
  "motion_id": "mot_8f2a91",
  "state": "MOVING",
  "estimated_duration_s": 2.4
}
```

**Errors**

| Status | Code | When |
|---|---|---|
| `400` | `INVALID_REQUEST` | Body is malformed, or `targets_deg` does not have exactly 6 values. |
| `409` | `ARM_NOT_READY` | Arm is in `DISABLED`, `FAULTED`, or `ESTOP` state. |
| `409` | `MOTION_IN_PROGRESS` | Another motion is running. Call `/motion/stop` first. |
| `422` | `JOINT_LIMIT_EXCEEDED` | A target is outside its joint range. |

---

### `POST /motion/stop`

Brings all joints to a controlled stop, decelerating along the planned path. This is a software stop. It is **not** a substitute for the hardware emergency stop.

**Response `200 OK`**

```json
{ "state": "READY", "stopped_motion_id": "mot_8f2a91" }
```

---

### `GET /faults`

Lists faults.

**Query parameters**

| Name | Type | Default | Description |
|---|---|---|---|
| `active` | boolean | `true` | `true` returns active faults only. `false` also returns faults cleared in the last 24 hours. |
| `joint` | integer | — | Filter by joint ID. |

**Response `200 OK`**: an array of [fault objects](#fault-object).

---

### `POST /faults/clear`

Clears all latched faults whose cause is no longer present and whose `clearable` field is `true`. Faults that are still present, or that need a physical reset, remain active.

**Response `200 OK`**

```json
{
  "cleared": ["flt_00412"],
  "still_active": ["flt_00413"],
  "state": "DISABLED"
}
```

After clearing, the arm stays `DISABLED`. Re-enabling the drives is a separate, deliberate call to [`POST /drives/enable`](#post-drivesenable). The controller never re-enables motors automatically.

---

### Standard error format

Every non-2xx response uses the same body:

```json
{
  "error": {
    "code": "JOINT_LIMIT_EXCEEDED",
    "message": "Target for joint 2 (135.0°) exceeds limit of 120.0°.",
    "joint": 2,
    "details": { "requested_deg": 135.0, "limit_deg": 120.0 },
    "request_id": "req_51c0de"
  }
}
```

| Status | Meaning in this API |
|---|---|
| `200` | Request succeeded. |
| `202` | Motion accepted and started. |
| `400` | Request is malformed. |
| `401` | Missing or invalid token. |
| `404` | Resource (for example, a joint ID) does not exist. |
| `409` | Request conflicts with the arm's current state. |
| `422` | Request is well-formed but physically invalid (for example, outside joint limits). |
| `503` | Controller cannot reach the drive bus. |

---

## 2. Real-Time Failure & Error State Specification

### Arm state machine

| State | Drives powered | Motion allowed | Entered when |
|---|---|---|---|
| `DISABLED` | No | No | At boot, after faults are cleared, or on request. |
| `READY` | Yes | Yes | Drives enabled with no active faults. |
| `MOVING` | Yes | Running | A motion command is executing. |
| `DERATED` | Yes | Yes, at reduced speed | A warning-level fault is active (for example, `THERMAL_THROTTLE`). |
| `FAULTED` | No | No | A fault-level condition disabled the drives. |
| `ESTOP` | No | No | The hardware emergency stop is engaged. |

These six are the **only** valid values of `state`. Clients should treat any other value as a protocol error.

### Fault object

```json
{
  "id": "flt_00413",
  "code": "OVER_CURRENT",
  "severity": "fault",
  "joint": 2,
  "detected_at": "2026-10-16T14:05:37.910Z",
  "measured": { "value": 9.6, "unit": "A" },
  "threshold": { "value": 8.0, "unit": "A" },
  "state_effect": "FAULTED",
  "active": true,
  "clearable": true,
  "action": "Check joint 2 for a collision or mechanical bind, then clear faults and re-enable."
}
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique fault instance ID. |
| `code` | string | Fault type. See the table below. |
| `severity` | string | `warning`, `fault`, or `critical`. |
| `joint` | integer or `null` | Affected joint, or `null` for arm-wide faults. |
| `measured` / `threshold` | object | Observed value and the limit it crossed, with units. |
| `state_effect` | string | The arm state this fault forces. |
| `active` | boolean | `true` while the fault is latched. |
| `clearable` | boolean | `true` if `/faults/clear` can clear it once the cause is gone. `false` for warnings, which clear themselves. |
| `action` | string | Recommended next step for the operator. |

### Fault codes

| Code | Severity | Trigger | Controller response | How it clears |
|---|---|---|---|---|
| `THERMAL_THROTTLE` | warning | Joint motor winding ≥ 70 °C | `velocity_override` reduced to 0.5; state → `DERATED`. | Automatically below 65 °C (`clearable: false`, no client action) |
| `OVER_TEMPERATURE` | fault | Joint motor winding ≥ 85 °C | Drives disabled; state → `FAULTED`. | `/faults/clear`, once below 65 °C |
| `ENCODER_SLIP` | fault | Motor and output encoders differ by > 0.5° | Drives disabled; state → `FAULTED`. Position can no longer be trusted. | `/faults/clear`, after re-homing the joint |
| `ENCODER_COMM_LOSS` | fault | No valid encoder frame for 5 ms | Drives disabled; state → `FAULTED`. | `/faults/clear`, once communication returns |
| `PEAK_OVER_CURRENT` | fault | Instantaneous drive current > 15.0 A (short circuit or hard collision) | Drive hardware cuts power within microseconds, independent of the controller; fault is **latched**; state → `FAULTED`. | `/faults/clear`, after inspecting the joint and wiring |
| `OVER_CURRENT` | fault | Drive current > 8.0 A continuously for > 10 ms (jam or overload) | Controller disables all drives; fault is **latched**; state → `FAULTED`. | `/faults/clear`, manual only |
| `ESTOP_ENGAGED` | critical | Hardware e-stop circuit opened | All drives off; state → `ESTOP`. | `/faults/clear` works only after the physical button is released and the safety circuit is reset |

### Key behaviours

- **Warnings derate. Faults stop.** A `warning` never disables the drives. A `fault` or `critical` always does.
- **Faults latch.** A fault stays active after its cause disappears, until a client calls `/faults/clear`. This prevents a joint from moving again without anyone noticing why it stopped.
- **No automatic re-enable.** Clearing a fault returns the arm to `DISABLED`, never directly to `READY`. Only `POST /drives/enable` powers the drives again.
- **Two layers of current protection.** `PEAK_OVER_CURRENT` is a hardware trip in the drive itself, for events too fast for software. `OVER_CURRENT` is the controller's sustained-overload check. Both latch.
- **Thermal throttling is the one exception** to latching. It is a warning, and it restores itself once the motor cools, with a 5 °C hysteresis band so it doesn't flicker on and off.

### How faults reach the client

Faults appear in three places:

1. `GET /faults`, for polling.
2. `GET /status`, through `state` and `active_faults`.
3. Any rejected request. For example, `POST /motion/joint` during a fault returns:

```json
{
  "error": {
    "code": "ARM_NOT_READY",
    "message": "Arm is FAULTED. Active fault: OVER_CURRENT on joint 2.",
    "details": { "state": "FAULTED", "active_fault_ids": ["flt_00413"] },
    "request_id": "req_7720ab"
  }
}
```

---

## 3. Sample Evaluation: Two Candidate Documentation Sets

### The task given to both models

> Using the ArmLink specification, document how the API reports an over-current fault on joint 2, including an example `GET /faults` response and what the client should do next.

### Candidate A (excerpt)

> When joint 2's drive current exceeds 8.0 A for more than 10 ms, the drive cuts power and the controller latches an `OVER_CURRENT` fault. The arm moves to `FAULTED` and all drives are disabled.
>
> ```json
> [{
>   "id": "flt_00413", "code": "OVER_CURRENT", "severity": "fault",
>   "joint": 2, "measured": { "value": 9.6, "unit": "A" },
>   "threshold": { "value": 8.0, "unit": "A" },
>   "state_effect": "FAULTED", "active": true, "clearable": true
> }]
> ```
>
> Inspect joint 2 for a collision or bind. Then call `POST /faults/clear`. The arm returns to `DISABLED`, and you must call `POST /drives/enable` before sending motion.

### Candidate B (excerpt)

> When joint 2 draws too much current, the controller enters the `PAUSED` state and reduces torque on that joint. Once current falls back below the limit, the joint re-enables automatically and motion resumes.
>
> ```json
> {
>   "fault": "OVER_CURRENT",
>   "joint_id": "J2",
>   "current": 9.6,
>   "state": "PAUSED"
> }
> ```
>
> `GET /faults` returns `200 OK` with this object. No client action is needed unless the fault repeats.

### Scoring rubric

| Criterion | Weight | Candidate A | Candidate B |
|---|---|---|---|
| Technical accuracy | 30% | 5 | 1 |
| Schema conformance | 25% | 5 | 1 |
| Safety correctness | 25% | 5 | 0 |
| Completeness | 10% | 4 | 2 |
| Clarity | 10% | 5 | 4 |
| **Weighted score (out of 5)** | | **4.90** | **1.15** |

### Issues found in Candidate B

| # | Issue | Why it is wrong | Severity |
|---|---|---|---|
| 1 | State `PAUSED` | Not one of the six defined states. The model invented a hardware state. | Critical |
| 2 | Joint "re-enables automatically and motion resumes" | Over-current faults **latch** and never auto-recover. Clearing returns the arm to `DISABLED`. Telling users a jammed joint restarts on its own is a safety hazard. | Critical |
| 3 | "Reduces torque" | Over-current protection cuts drive power. It does not derate. B is describing thermal-throttle behaviour under the wrong fault. | Major |
| 4 | Payload shape | `GET /faults` returns an **array** of fault objects. B returns a single object with invented fields (`fault`, `current`) and the wrong joint type (`"J2"` instead of integer `2`). It also omits `id`, `severity`, `threshold`, `active`, and `clearable`. | Major |
| 5 | "No client action is needed" | A latched fault requires `POST /faults/clear` and then `POST /drives/enable`. Client code written from B would wait forever. | Major |

### Minor note on Candidate A

The excerpt leaves out the `detected_at` and `action` fields from the example payload. That is acceptable in a short excerpt, but the full reference should show the complete object.

### Verdict

**Preferred: Candidate A.**

**Rationale:** B invents a `PAUSED` state that does not exist and claims an over-current fault auto-recovers. That is the opposite of the latching behaviour the spec requires, and it would lead integrators to write unsafe client code. A is accurate, matches the fault schema, and gives the correct clear-then-enable sequence.

### What this example shows

Candidate B reads smoothly and sounds confident. Its errors only show up when you check it against the state machine and fault table. For hardware APIs, documentation errors are not cosmetic: a wrong statement about recovery behaviour can lead someone to write code that moves a robot unexpectedly. Good evaluation means checking every state, field, and behaviour against the specification, not just judging the prose.
