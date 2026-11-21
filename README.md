# Star Citizen 5.0 PTU — Game Telemetry System
## Reverse-Engineering Documentation & MFD Integration Guide

**Target build:** `sc-alpha-5.0.0` / `5.0.202.205` PTU, `Bin64\StarCitizen.exe` 190,220,288 bytes, built 2026-11-19.

---

## 1. TL;DR

Star Citizen 5.0 ships an **in-process gRPC server** ("local services server") that streams the local player's game state as **protobuf messages over HTTP/2 (plaintext, "h2c")**:

| Item | Value |
|---|---|
| Service | `sc.game.telemetry.v1.GameTelemetryService` |
| Main RPC | `TelemetryStream` — server-streaming, empty request, **~20 Hz** frames |
| Transport | gRPC 1.49.2 over cleartext HTTP/2 |
| Default port | **2942** (clamped 1024–65535) |
| Default bind | `127.0.0.1`; `0.0.0.0` when `grpc-client-server-external 1` |
| Auth | **None** — anything that can reach the port can read telemetry and fire allowlisted actions |
| Health check | standard `grpc.health.v1.Health/Check` is hosted |
| Server reflection | **not** hosted (use the .proto files in this repo) |

**Minimal enablement** (put in `PTU\user.cfg`, or type in the PTU console):

```
server-game-telemetry-enabled 1
grpc-client-server-external 1      // only if an ESP32/other LAN device connects
```

The game logs `Local services server listens on <ip>:<port>: other computers on the network can read telemetry and trigger game actions` when the LAN bind is active.

---

## 2. Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    StarCitizen.exe (client)                │
│                                                            │
│  CLocalPlayerTelemetryComponent ──┐                        │
│   (collects local player state:   │ 50 ms tick (default)   │
│    vehicle, actor, weapons,       ▼                        │
│    events, force-feedback)   CGameTelemetryServer          │
│                                   │  builds protobuf       │
│  SGameTelemetryConfig (cvars) ─────┤  TelemetryStreamResponse
│                                   ▼                        │
│                     CCIGServices local gRPC server         │
│                     ("services server", grpc 1.49.2)       │
│                     127.0.0.1:2942  (or 0.0.0.0:2942)      │
└────────────────────────────┬───────────────────────────────┘
                             │  HTTP/2 (h2c) + protobuf
              ┌──────────────┴───────────────┐
              ▼                              ▼
     PC tools (grpcurl,                      ESP32-P4 MFD
     inspector/bridge)                       (this repo)
```

* The telemetry is **client-side**: it reflects *your* local player. The `server-` prefix on the cvars is historical naming (they exist for the dedicated-server build too).
* `grpc-client-server-enabled 1` hosts the server at launch regardless of telemetry; enabling `server-game-telemetry-enabled` **starts the server on demand** ("Turning it on starts that server; turning it off stops it if telemetry started it").
* The same server also hosts `grpc.health.v1.Health` and (test) echo services; game telemetry registers under service name **`game_telemetry`** in the services config.

---

## 3. Configuration (console variables)

All names exist in three forms: kebab-case console/user.cfg name, `SCREAMING_SNAKE` env var, and dotted config key. Defaults verified in code (`SGameTelemetryConfig` registration @ `0x14636fb00`, server start @ `0x1463e0390`).

### 3.1 Game telemetry (`SGameTelemetryConfig`)

| cvar (user.cfg / console) | env / config key | default | meaning |
|---|---|---|---|
| `server-game-telemetry-enabled` | `SERVER_GAME_TELEMETRY_ENABLED` / `server.game.telemetry.enabled` | **0** | Serve telemetry on the local services server. Enabling starts the server; disabling stops it *if telemetry started it*. |
| `server-game-telemetry-test-data` | `server.game.telemetry.test_data` | **0** | Synthetic test data instead of real game state — **use this to bench-test your MFD without launching into the verse**. |
| `server-game-telemetry-stream-interval-ms` | `server.game.telemetry.stream.interval.ms` | **50** (≈20 Hz) | Frame interval; clamped ≥ 1. |
| `server-game-telemetry-generate-without-clients` | `server.game.telemetry.generate.without.clients` | **0** | Generate frames even with no connected client (keeps data fresh / warms the pipe). |
| `server-game-telemetry-trigger-actions-enabled` | `server.game.telemetry.trigger.actions.enabled` | **1** | Allow external clients to trigger allowlisted actions. |
| `server-game-telemetry-trigger-debounce-ms` | `server.game.telemetry.trigger.debounce.ms` | **200** | Min interval between repeats of the same action. |

### 3.2 Local gRPC server (`SServiceConfigGrpcClientServer`)

| cvar | env / config key | default | meaning |
|---|---|---|---|
| `grpc-client-server-enabled` | `GRPC_CLIENT_SERVER_ENABLED` / `grpc.client.server.enabled` | 0 | Host the local services server at launch. |
| `grpc-client-server-port` | `GRPC_CLIENT_SERVER_PORT` / `grpc.client.server.port` | **2942** | "Port the client-hosted gRPC server binds to, 1024 to 65535". |
| `grpc-client-server-external` | `GRPC_CLIENT_SERVER_EXTERNAL` / `grpc.client.server.external` | 0 | "Bind to 0.0.0.0 to accept connections from other computers, instead of 127.0.0.1 for this computer only." |
| `grpc-client-server-channel-args` | `grpc.client.server.channel.args` | "" | "JSON object of gRPC server channel args (grpc.* keys, string or int values)." |

Console command: **`gt_trigger <action_id> [hold_ms]`** — fires an allowlisted action locally (same path `TriggerGameAction` uses). Handy for testing the allowlist.

### 3.3 Recommended `user.cfg` for the ESP32 MFD

```ini
// PTU\user.cfg
server-game-telemetry-enabled 1
grpc-client-server-external 1
// optional tuning:
server-game-telemetry-stream-interval-ms 50
server-game-telemetry-trigger-actions-enabled 1
// while building the display (no real flight data needed):
// server-game-telemetry-test-data 1
```

> ⚠️ **Security note:** with `external 1`, *anyone on your LAN* can read your telemetry (incl. your aUEC balance) and trigger allowlisted ship actions. Keep it on a trusted network, turn it off when not in use.

---

## 4. Wire protocol

* Plain **gRPC over cleartext HTTP/2** (h2c, prior-knowledge — no `Upgrade:` dance). No TLS configuration exists on the local server path (no credentials/keys involved; the `grpc-server-transport-security` cvar belongs to the *backend* `grpc.server.*` client config group).
* Paths:
  * `/sc.game.telemetry.v1.GameTelemetryService/TelemetryStream`
  * `/sc.game.telemetry.v1.GameTelemetryService/TriggerGameAction`
  * `/sc.game.telemetry.v1.GameTelemetryService/ListTriggerableActions`
  * `/grpc.health.v1.Health/Check`
* Message framing: standard gRPC — 5-byte prefix (`0x00` compressed-flag + big-endian u32 length) + protobuf payload, inside HTTP/2 DATA frames.
* `TelemetryStream` is **server-streaming**: send the (empty) request with END_STREAM, then the server pushes `TelemetryStreamResponse` at the configured interval for every connected client.
* Quick connectivity probes:
  * TCP: `Test-NetConnection <game-pc-ip> -Port 2942` (PowerShell)
  * gRPC health: `grpcurl -plaintext <ip>:2942 grpc.health.v1.Health/Check`
  * Full stream: `grpcurl -plaintext -d '{}' -proto api.proto -import-path . <ip>:2942 sc.game.telemetry.v1.GameTelemetryService/TelemetryStream`

---

## 5. Protobuf schema

Reconstructed **exactly** from the `FileDescriptorProto` blobs embedded in the exe (`sc/game/telemetry/v1/types.proto`, `sc/game/telemetry/v1/api.proto`). Canonical files: [`proto/sc/game/telemetry/v1/types.proto`](../proto/sc/game/telemetry/v1/types.proto), [`proto/sc/game/telemetry/v1/api.proto`](../proto/sc/game/telemetry/v1/api.proto). Raw descriptor dumps: `proto/*.bin`.

### 5.1 Service

```protobuf
service GameTelemetryService {
  rpc TelemetryStream (TelemetryStreamRequest) returns (stream TelemetryStreamResponse);
  rpc TriggerGameAction (TriggerGameActionRequest) returns (TriggerGameActionResponse);
  rpc ListTriggerableActions (ListTriggerableActionsRequest) returns (ListTriggerableActionsResponse);
}
```

### 5.2 The stream frame

```protobuf
message TelemetryStreamResponse {
  VehicleFrame vehicle = 3;              // when seated in / flying a vehicle
  WeaponsFrame weapons = 4;              // when piloting
  EventsFrame events = 5;                // discrete events (spawn/dead/seat changes)
  ActorFrame actor = 6;                  // always (local player)
  ForceFeedbackFrame force_feedback = 7; // motion/haptic (D-BOX style)
  TurretFrame turret = 8;                // when in a turret
}
```

Fields absent in a frame are simply not set (proto3): use presence, not zero-values, to decide what to render.

### 5.3 Highlights (full details in the .proto files)

* **VehicleState** — ship name/class/manufacturer/type, hull %, lights, doors (open/lock), **annunciator panel** (`pwr wpn thr shld cool qdrv radr prox hit misl`, each `NOMINAL/OFF/MINOR_FLASHING/MINOR_ON/MAJOR_FLASHING/MAJOR_ON` — this is your MFD caution/warning panel), powerplant bitfield, shields (overall + per-face), master mode (NAV/SCM/STEALTH + spooled).
* **VehicleFlight.instruments** — `speed, hydrogen_fuel, quantum_fuel, g_meter(current,max), boost(engaged/output/reserves/critical), altitude, radar_altitude, vertical_speed_indicator, atmospheric_density, heading`.
* **VehicleFlight.settings** — full IFCS state as `Availability{enabled,active}`: coupled, ESP, gravity-compensation, G-safe, cruise, speed-limiter, proximity-assist (+landing), stability, core, wind-compensation, command-behaviour.
* **VehicleFlight.state** — `is_landed`, gear/VTOL/transformation as `RETRACTED/DEPLOYING/DEPLOYED/RETRACTING`.
* **ActorVitals** — health/oxygen/hunger/thirst/body-temperature gauges, **heart rate**, gravity, pressure, intoxication.
* **ActorState** — locomotion (ON_FOOT/EVA/SEATED), crew role (PILOT/GUNNER/CREW), seat name (`pilot_seat`, `turret_top`, `copilot_seat`, …), **aUEC balance**.
* **ActorEnvironment** — external temperature, radiation rate, breathable atmosphere.
* **WeaponsFrame** — weapon preset, per-weapon powered/ready flags, countermeasures (decoy/noise counts), weapon settings (lag pips, gimbals locked, …).
* **EventsFrame** — ACTOR_SPAWN / DEAD / ENTER_SEAT / LEAVE_SEAT / RESPAWN / DOWNED.
* **ForceFeedbackFrame** — per-axis (x/y/z) resistance & damping, linear + angular, gunfire, stick-shaker.

> Removed-field trivia: `VehicleFrame` has no field 1, `TelemetryStreamResponse` starts at 3, `ForceFeedbackFrame` at 2, `ActorVitals` skips 6 — fields were deleted during development (schema is not field-dense; don't assume holes are "reserved position").

---

## 6. TriggerGameAction — the allowlist

External clients can fire **allowlisted** game actions (MFD soft-buttons!). Status codes: `OK, UNKNOWN_ACTION, MAP_INACTIVE, RATE_LIMITED, FAILED, HOLD_NOT_ALLOWED, FILTERED`.

The allowlist is a static table at VA `0x1498139B8–0x149814A20` (entries: `{const char* actionId; const char* actionMap; uint32 code; …}`, stride 0x28). **106 entries** fully extracted (see `ghidra_out/allowlist.txt`):

**player_emotes (34)** — `emote_agree, emote_angry, emote_atease, emote_attention, emote_blah, emote_bored, emote_bow, emote_burp, emote_cheer, emote_chicken, emote_clap, emote_come, emote_cry, emote_dance, emote_disagree, emote_failure, emote_flex, emote_flirt, emote_gasp, emote_gloat, emote_greet, emote_laugh, emote_launch, emote_point, emote_rude, emote_salute, emote_sit, emote_sleep, emote_smell, emote_taunt, emote_threaten, emote_wait, emote_wave, emote_whistle`

**seat_general (17)** — `v_eject, v_emergency_exit, v_enter_remote_turret_1/2/3, v_light_amplification_off/on/toggle, v_operator_mode_cycle_back/forward, v_set_flight_mode, v_set_guns_mode, v_set_mining_mode, v_set_quantum_mode, v_set_refuel_mode, v_set_salvage_mode, v_set_scan_mode`

**spaceship_docking (1)** — `v_toggle_docking_request`

**spaceship_general (11)** — `v_close_all_doors, v_flightready, v_lock_all_doors, v_lock_all_ports, v_open_all_doors, v_self_destruct, v_toggle_all_doorlocks, v_toggle_all_doors, v_toggle_all_portlocks, v_unlock_all_doors, v_unlock_all_ports`

**spaceship_movement (27)** — `v_atc_request, v_toggle_landing_system, v_deploy_landing_system, v_retract_landing_system, v_flight_advanced_hud_off/on/toggle, v_ifcs_toggle_gforce_safety, v_ifcs_gsafe_on/off, v_ifcs_proximity_assist_off/on/toggle, v_ifcs_reset_gmeter_max, v_ifcs_toggle_esp, v_ifcs_vector_decoupling_off/on/toggle, v_master_mode_cycle, v_master_mode_set_nav, v_master_mode_set_scm, v_transform_cycle/deploy/retract, v_vtol_off/on/toggle`

**spaceship_power (10)** — `v_power_set_off/on, v_power_set_shields_off/on, v_power_set_thrusters_off/on, v_power_set_weapons_off/on, v_power_toggle, v_power_toggle_shields, v_power_toggle_thrusters, v_power_toggle_weapons`

**spaceship_quantum (1)** — `v_toggle_qdrive_engagement`

**spaceship_radar (1)** — `v_invoke_ping`

**spaceship_target_hailing (1)** — `v_target_hail`

Notably **weapons are excluded** (the shipped unit test asserts `FindTriggerableAction("v_fire_weapon_group_1") == nullptr`), while `V_TOGGLE_LANDING_SYSTEM` *is* present. Triggering is additionally filtered by actor/seat context (`ActorAndSeatPreference` — seat names like `pilot_seat`, `turret_top`, `copilot_seat` appear in the test strings) and returns `MAP_INACTIVE` when the relevant action map isn't active. Prefer `ListTriggerableActions` at runtime for the live list with `category` + `holdable`.

`TriggerGameActionRequest { string action_name; optional uint32 hold_ms; }` — holdable actions accept a hold duration. Actions must be legal in the current context (`MAP_INACTIVE` when the relevant action map — e.g. `spaceship_movement` — isn't active, i.e. you're not flying).
