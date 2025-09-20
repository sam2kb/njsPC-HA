## njsPC-HA – AI Coding Agent Instructions

Purpose: Enable fast, correct contributions to this Home Assistant custom integration that bridges nodejs-PoolController (njsPC) to Home Assistant (HA) via native HTTP + Socket.IO (no MQTT).

### High-Level Architecture
1. Integration domain: `njspc_ha` (see `manifest.json`). Push (not poll) – `iot_class: local_push`.
2. Connection lifecycle:
   - Config flow (`config_flow.py`) discovers host/port (Zeroconf + SSDP) or uses manual input.
   - On setup (`__init__.py`):
     * Fetch full initial state via HTTP: `GET /state/all` (constant `API_STATE_ALL`).
     * Instantiate `NjsPCHAapi` (thin HTTP wrapper) + `NjsPCHAdata` (a `DataUpdateCoordinator`).
     * Open persistent `socketio.AsyncClient` to njsPC base URL.
   - Socket.IO event names map 1:1 to equipment categories (see `const.py`). Each handler stamps `data["event"]` then calls `async_set_updated_data()` and forwards the payload onto the HA event bus as `njspc-ha_event` with structure `{evt, data}`.
3. Entity model:
   - All entities subclass `PoolEquipmentEntity` (in `entity.py`) which standardizes: unique ids, `DeviceInfo` grouping, naming conventions, availability handling, and formatting helpers.
   - Equipment classes + human models enumerated in `PoolEquipmentClass` & `PoolEquipmentModel` (in `const.py`) and mapped via `DEVICE_MAPPING`.
4. Platforms implemented: sensor, binary_sensor, climate, number, light, button, switch. Each platform file builds entities once from the initial config object, then relies entirely on pushed events.
5. Command pathway: Write actions always use HTTP PUT to specific endpoints (constants in `const.py` such as `API_CIRCUIT_SETSTATE`, `API_SET_HEATMODE`, etc.) via `NjsPCHAapi.command(url, data)`.

### Event & Update Pattern
- Every Socket.IO handler (defined inline in `NjsPCHAdata.sio_connect`) attaches an `event` key then dispatches.
- Entity instances implement `_handle_coordinator_update()` (called by `CoordinatorEntity`) to filter on `self.coordinator.data["event"]` and equipment id, updating internal state and calling `async_write_ha_state()`.
- Availability is synthesized: on socket connect => `EVENT_AVAILABILITY` with `available: True`; on disconnect or connection errors => `available: False`.

### Key Conventions
1. Unique IDs: `f"{controller_id}_{equipment_class}_{equipment_id}[ _<suffix>]"`. `controller_id` = host with dots removed + port.
2. Device aggregation: Each physical/logical equipment piece becomes a HA device keyed by tuple `(DOMAIN, model, equipment_class, equipment_id)` so multiple entities share a device card.
3. Do NOT poll — all entities set `should_poll = False`. Any new entity must follow push-only semantics.
4. Guard for partial payloads: Events may omit fields; entity handlers must only overwrite attributes when keys are present (pattern already used widely – mirror it).
5. Light effects: Light themes fetched at startup (`get_lightthemes`) and stored as map `val -> desc`; effect selection translates HA effect string back to original `val` when sending `API_CIRCUIT_SETTHEME`.
6. Climate (heater) logic: If only two heat modes (off + one heat source) => use HVAC modes (OFF + HEAT / HEAT_COOL). More than two => expose a single HVAC mode AUTO with named presets derived from heat modes.
7. Chemistry & chlorinator: Many numerical / binary entities rely on nested dictionaries containing `val`, `desc`; warnings/alarms aggregate multiple subfields by summing integer flags. Maintain attribute naming consistency (snake_case, descriptive) as seen in `chemistry.py`.
8. Schedules: `ScheduleSwitch` normalizes day bitmasks into human-readable strings and start/end times with respect to 12/24h mode supplied in initial config. Preserve formatting helpers when extending.
9. Coverage sensors: `BodyCoveredSensor.is_on` intentionally returns logical NOT of `isCovered` to represent “cover open” as on — replicate this inversion if adding related sensors.

### Adding New Functionality
- For a new equipment type received via a new Socket.IO event:
  1. Add constant for event name & any endpoints to `const.py`.
  2. Register a handler inside `NjsPCHAdata.sio_connect` mirroring existing ones.
  3. Extend `PoolEquipmentClass` / `PoolEquipmentModel` and `DEVICE_MAPPING` if it must appear as its own HA device.
  4. Create platform entities (re-use an existing platform file if category fits; otherwise add a new file & include platform in `PLATFORMS` list in `__init__.py`).
  5. Follow unique id + availability conventions. Avoid synchronous/blocking IO in entity code.

### API / Data Shape Assumptions
- The initial config (from `/state/all`) contains arrays: `circuits`, `pumps`, `filters`, `heaters`, `chlorinators`, `chemControllers`, `lightGroups`, `circuitGroups`, `features`, `schedules`, `temps` (with `bodies`). Subsequent events deliver deltas (not full snapshots). Code must tolerate missing keys.
- Some derived values (e.g., pump program) depend on bitwise/log math (see `PumpProgramSensor.get_program`). Preserve logic when refactoring; add unit tests if altering.

### Error Handling & Logging
- Network / HTTP failures log at `_LOGGER.error` with response text; minimal exception raising (integration favors resilience). New commands should match this pattern instead of raising.
- Suppress noisy Socket.IO internal logs (explicit level overrides in `sio_connect`). Do not remove those overrides.

### Home Assistant Integration Details
- `manifest.json`: lists single external dependency `python-socketio[asyncio_client]`. Do not add heavy dependencies; prefer stdlib or HA helpers.
- Config flow uniqueness enforced via composed `server_id` (`njspcha_<host-without-dots><port>`). Maintain this rule to avoid duplicate entries.
- Zeroconf type `_poolcontroller._tcp.local.` and SSDP deviceType `urn:schemas-tagyoureit-org:device:PoolController:1` — update both if discovery protocol evolves.

### Common Gotchas
- Do not assume Fahrenheit; temperature units come from `config["temps"]["units"]["name"]` or per-event `units` key.
- Some systems may lack optional arrays (e.g., no heaters, no virtual circuits). Always iterate defensively.
- Chemistry controller fields can be absent or disabled (`enabled` flag). Add entities only when enabled; mirror current conditional checks.
- Light group commands / themes fetched asynchronously at setup; avoid blocking startup by adding long serial loops — prefer gathering concurrently if expansion needed.

### Testing / Validation Hints
- Manual: Start HA with integration pointed at a running nodejs-PoolController; toggle circuits in dashPanel and confirm entity state updates instantly (no polling delays).
- For new event types: Temporarily log raw event payload to ensure structure before mapping to attributes.

### When Unsure
- Search for existing pattern in analogous entity class (e.g., see `_handle_coordinator_update` usage). Consistency is higher priority than consolidation unless a clear cross-entity abstraction reduces duplication without runtime cost.

---
Feedback: If any section seems incomplete or you need deeper details (e.g., adding tests, refactoring for typing), ask and we can iterate.