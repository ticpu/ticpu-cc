# ESL Event Reference

## Event Header Naming Convention

FreeSWITCH serializes caller profile fields into event headers by
prepending `Caller-` to the field name. Since some caller profile fields
already start with `Caller-`, this produces double-prefixed headers:

| Caller Profile Field | Event Header |
|---|---|
| Caller-ID-Number | `Caller-Caller-ID-Number` |
| Caller-ID-Name | `Caller-Caller-ID-Name` |
| ANI | `Caller-ANI` |
| Destination-Number | `Caller-Destination-Number` |
| Channel-Name | `Caller-Channel-Name` |
| Unique-ID | `Caller-Unique-ID` |

Note: `Caller-ANI` is distinct from `Caller-Caller-ID-Number`. ANI
(Automatic Number Identification) is network-level billing identification;
Caller ID is user-facing presentation. In SIP they often carry the same
value, but FreeSWITCH maintains them as separate caller profile fields.

Source: `switch_event_prep_for_delivery()` in `src/switch_event.c`.

## Common Event Headers

| Header | Description |
|---|---|
| `Event-Name` | Wire name (e.g. `CHANNEL_CREATE`, `CUSTOM`) |
| `Event-Subclass` | Subclass for CUSTOM events (e.g. `sofia::register`) |
| `Unique-ID` | Channel UUID |
| `Channel-Name` | Channel identifier (e.g. `sofia/internal/1000@domain`) |
| `Channel-State` | Current channel state (e.g. `CS_EXECUTE`) |
| `Channel-Call-State` | Call state (e.g. `ACTIVE`, `RINGING`) |
| `Hangup-Cause` | Q.850 hangup cause name |
| `Job-UUID` | UUID for bgapi BACKGROUND_JOB events |
| `variable_*` | Channel variables (prefixed with `variable_`) |

## CUSTOM Event Subclasses

CUSTOM events use `Event-Subclass` with `module::event` naming:

### Sofia (mod_sofia)

- `sofia::register` — SIP endpoint registered
- `sofia::unregister` — SIP endpoint unregistered
- `sofia::expire` — Registration expired
- `sofia::register_attempt` — Registration attempt (before auth)
- `sofia::register_failure` — Registration auth failed
- `sofia::gateway_add` — Gateway added to profile
- `sofia::gateway_delete` — Gateway removed
- `sofia::gateway_state` — Gateway state change (UP/DOWN/TRYING)
- `sofia::pre_register` — Pre-registration hook
- `sofia::notify_refer` — SIP REFER notification

### Conference (mod_conference)

- `conference::maintenance` — Conference state changes (member join/leave,
  mute, floor changes, etc.)

### Other Modules

- `callcenter::info` — Call center queue events
- `fifo::info` — FIFO queue events
- `valet_parking::info` — Valet parking events

## Event Priority

FreeSWITCH events carry a `priority` header with values `NORMAL`, `LOW`,
or `HIGH` (matching `switch_priority_t` / `esl_priority_t`).

**Priority is metadata only.** The event dispatch system delivers events
FIFO — priority does not affect ordering. The `switch_event_deliver()`
function processes events from a FIFO queue without sorting by priority.

Priority is:
- Stored in the event's `priority` header when sent via `sendevent`
- Preserved through serialization/deserialization
- Available for consumers to read and filter on
- **Not** used by core event dispatch for ordering

## HEARTBEAT Event

The global HEARTBEAT event is fired by a scheduler task at a fixed
interval. It contains system health metrics (CPU, sessions, uptime).

| Aspect | Details |
|---|---|
| Config parameter | `event-heartbeat-interval` in `switch.conf.xml` `<settings>` |
| C struct field | `runtime.event_heartbeat_interval` (seconds) |
| Default interval | 20 seconds |
| Runtime changeable | No — config only read at startup, no API to modify |
| Scope | Global (single scheduler task, all ESL connections) |
| Source | `heartbeat_callback()` in `src/switch_core.c` |

```xml
<!-- switch.conf.xml -->
<param name="event-heartbeat-interval" value="20"/>
```

Note: `SWITCH_EVENT_SESSION_HEARTBEAT` is a separate per-channel event
controlled via `uuid_session_heartbeat` API, unrelated to the global
HEARTBEAT used for ESL liveness detection.

## Common Hangup Causes

Q.850 cause names as used by FreeSWITCH (`switch_call_cause_t` in
`src/include/switch_types.h`):

| Cause | Q.850 Code | When |
|---|---|---|
| `NORMAL_CLEARING` | 16 | Normal call hangup (most common) |
| `USER_BUSY` | 17 | Callee is busy (486 SIP) |
| `NO_USER_RESPONSE` | 18 | No response from endpoint |
| `NO_ANSWER` | 19 | Rang but not answered |
| `CALL_REJECTED` | 21 | Call refused (403/603 SIP) |
| `UNALLOCATED_NUMBER` | 1 | Number not found (404 SIP) |
| `NORMAL_TEMPORARY_FAILURE` | 41 | Temporary failure (500 SIP) |
| `ORIGINATOR_CANCEL` | FS-specific | Caller hung up before answer |
| `LOSE_RACE` | FS-specific | Lost in forked dialing (parallel ring) |
| `RECOVERY_ON_TIMER_EXPIRE` | 102 | Timer expired during recovery |
| `NORMAL_UNSPECIFIED` | 31 | Generic normal-class hangup |
| `NETWORK_OUT_OF_ORDER` | 38 | Network failure |
| `DESTINATION_OUT_OF_ORDER` | 27 | Destination unreachable |

### SIP Status Code Mapping

FreeSWITCH maps between SIP status codes and Q.850 causes. The mapping
is defined in `src/switch_core.c`. Key mappings:

| SIP | Q.850 Cause |
|---|---|
| 404 | `UNALLOCATED_NUMBER` |
| 486 | `USER_BUSY` |
| 408 | `NO_USER_RESPONSE` / `RECOVERY_ON_TIMER_EXPIRE` |
| 480 | `NO_ANSWER` |
| 487 | `ORIGINATOR_CANCEL` |
| 403/603 | `CALL_REJECTED` |
| 500 | `NORMAL_TEMPORARY_FAILURE` |
| 503 | `SWITCH_CONGESTION` |

### Related Variables

- `hangup_cause` — Current channel's hangup cause
- `bridge_hangup_cause` — B-leg's hangup cause after bridge
- `proto_specific_hangup_cause` — SIP status or ISDN cause code
- `sip_term_cause` — Q.850 code from termination
- `sip_ignore_remote_cause` — Ignore remote hangup cause
- `continue_on_fail` — Comma-separated list of causes to retry on

## Default Originate Timeout

When no timeout is specified in the originate command and the
`originate_timeout` channel variable is not set, the default is **60
seconds** (`SWITCH_DEFAULT_TIMEOUT` in `src/include/switch_types.h`).

The `call_timeout` variable can also affect individual leg timeouts,
and `leg_timeout` sets per-endpoint timeouts in forked dialing.
