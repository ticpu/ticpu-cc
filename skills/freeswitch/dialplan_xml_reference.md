# FreeSWITCH XML Dialplan Implementation Reference

## Architecture
**Hunt Phase**: Conditions evaluated, actions queued
**Execute Phase**: Actions executed sequentially
**Critical**: Variables set in execute phase unavailable to hunt phase conditions

**Limits**: 100 nested conditions max, 10 PCRE capture groups max
**Context**: Exact match or "global" fallback, no inheritance
**Order**: Document order unless `auto_hunt` used

## Conditions
**Break**: BREAK_ON_FALSE (default), BREAK_ON_TRUE, BREAK_ALWAYS, BREAK_NEVER
**Field resolution**: `$` → channel var, else caller profile field, else empty
**Time**: `switch_xml_std_datetime_check()` returns 1/0/-1, supports `tod_tz_offset`/`timezone`
**Multi-regex**: `all` (AND), `any` (OR), `xor` (XOR)
**Captures**: `$1-$10`, `DP_MATCH`, `DP_REGEX_MATCH_N` - condition-scoped only
**Nested**: `require-nested=true` (default) - parent fails if any nested fail
**Order**: Parent actions → remaining parent → nested breadth-first

## Applications

### Inline-Capable (SAF_ROUTING_EXEC) - Hunt Phase Execution
**Variables**:
- `set`/`multiset`: `base_set()` with `SWITCH_STACK_BOTTOM`. Variable expansion via `switch_channel_expand_variables()`. Immediate hash storage.
- `push`/`unshift`: Stack operations via `SWITCH_STACK_PUSH`/`SWITCH_STACK_UNSHIFT`. For multi-value handling.
- `set_global`: Core global variable via `switch_core_set_variable()`. Process-wide scope.
- `set_profile_var`: Profile-specific vars. Context: SIP profile settings.
- `unset`/`multiunset`: Hash removal via `switch_channel_set_variable(chan, var, NULL)`. Supports comma-delimited.
- `export`/`bridge_export`: Cross-bridge variable propagation. Sets `__` prefix for export scope.

**Control**:
- `eval`: Pure no-op. Only triggers variable expansion in data field. Zero processing overhead.
- `stop`: Alias to `eval`. Misleading name - doesn't stop execution.
- `check_acl`: ACL validation via `switch_check_network_list()`. Optional hangup with cause code.
- `set_user`: Directory user application. Loads user settings, applies to channel. Modifies caller profile.
- `verbose_events`: Sets `SWITCH_CHANNEL_LOG_CLEAN` flag. Affects event verbosity globally.

**Media Control**:
- `novideo`: Sets `CF_NOVIDEO` flag. Affects codec negotiation, refuses video streams.
- `cng_plc`: Sets `CF_CNG_PLC` flag. Enables PLC (Packet Loss Concealment) on CNG frames.
- `early_hangup`: Sets `CF_EARLY_HANGUP` flag. Allows hangup before answer.
- `presence`: Presence state modification via `switch_core_add_registration()`.

**External Integration**:
- `curl`: HTTP client via libcurl. Supports POST/GET, form data, authentication.
- `enum`: ENUM lookup via DNS. RFC 3761 NAPTR record processing.
- `easyroute`: Database-driven routing. Custom query execution.
- `lcr`: Least Cost Routing. Rate table lookups, cost calculations.
- `nibblebill`: Real-time billing. Account balance checks, debit operations.
- `cidlookup`: Caller ID database lookup. Name resolution from number.

### Execute-Only Applications

#### Audio/Media Processing
**answer**: 
- Sets `CF_ANSWERED` flag, transitions to `CS_MEDIA` state
- Starts RTP threads via `switch_core_session_start_audio_write_thread()`
- Optional flags: `is_conference` (CF_CONFERENCE), `decode_video` (CF_VIDEO_DECODED_READ)
- Non-blocking, immediate return after flag/state change

**pre_answer**:
- Sets `CF_EARLY_MEDIA` flag, enables early media without billing start
- Codec negotiation occurs, RTP established, no answer supervision sent
- Used for ringback, announcements before answer

**hangup**:
- Cause code via `switch_channel_str2cause()`. Default: `SWITCH_CAUSE_NORMAL_CLEARING`
- Calls `switch_channel_hangup()` - triggers state machine to `CS_HANGUP`
- Immediate effect: channel cleanup, CDR finalization, resource deallocation

**sleep**:
- Uses `switch_ivr_sleep()` with millisecond precision
- Optional DTMF interrupt via `sleep_eat_digits` variable
- `on_dtmf()` callback checks `playback_terminators` variable
- Interruptible sleep, not blocking system sleep

**playback**:
- File handle via `switch_ivr_play_file()`, supports offset (@@ syntax)
- `fh.samples` for position seeking. Auto-detects file format.
- DTMF callback `on_dtmf()`, terminator support via `playback_terminators`
- Sets `SWITCH_PLAYBACK_TERMINATOR_USED` variable on interrupt

**record**:
- File handle for recording, arguments: `<file> [<limit_seconds>] [<silence_thresh>] [<silence_hits>]`
- Silence detection via configurable thresholds and hit counts
- Sample rate from session, format auto-detected from extension
- File locking, handles concurrent access issues

#### Call Control & Routing
**bridge**:
- Complex campon logic: retries, timeouts, fallback extensions
- Music on hold during campon via configurable MOH source
- Thread-based campon with `camping_stake` structure for state tracking
- Caller ID propagation: `effective_caller_id_*` variables take precedence
- Bridge loop handles media relay, codec transcoding, DTMF relay
- Blocking operation until bridge completes or fails

**transfer**:
- Supports A-leg (`transfer ext`), B-leg (`transfer -bleg`), both (`transfer -both`)
- B-leg transfer requires partner UUID lookup via `switch_channel_get_partner_uuid()`
- Uses `switch_ivr_session_transfer()` - resets hunt phase, new context/extension
- Hunt phase restart: conditions re-evaluated from transfer target

**park**:
- Infinite loop via `switch_ivr_park()` - channel stays active indefinitely
- Periodic checks for channel state, transfer events
- CPU-efficient: yields between checks
- Exit conditions: transfer, hangup, or explicit unpark

#### Advanced Applications
**originate**:
- Creates new session via `switch_core_session_outgoing_channel()`
- Supports timeout, progress callbacks, channel variables
- Complex syntax: `<call_url> [<timeout>] [<caller_id_name>] [<caller_id_number>]`
- Early media handling, ringback generation
- UUID tracking for session management

**conference**:
- Room management via `conference_member_t` structures
- Audio mixing engine, multiple algorithms (mix, mux, etc.)
- Member controls: mute, deaf, kick, recording
- Video mixing for multi-party video calls
- MOH, announcements, caller controls (dial-out, etc.)

**fifo**:
- Queue management with priority levels (1-10)
- Consumer/producer model: agents consume, callers produce
- Callbacks on queue events (empty, full, timeout)
- Music on hold, announcements, position announcements
- Statistics tracking: calls waiting, agents available

#### Script Engine Integration
**lua**:
- Lua state creation per session via `switch_lua_init()`
- Session object binding: `session:*` methods available
- Script path resolution, module loading
- Error handling, script timeout protection

**javascript**:
- SpiderMonkey engine integration
- Session object with JavaScript bindings
- File system access, network operations
- Memory management, garbage collection

**python**:
- Python interpreter embedding
- Session object as Python class
- Module import capability
- Exception handling, stack trace capture

#### Database & External
**db**:
- Core database API via `switch_core_db_*()` functions
- Supports SQLite (embedded), PostgreSQL, MySQL via ODBC
- SQL injection protection via prepared statements
- Transaction support, connection pooling

### Advanced Application Behaviors

#### DTMF Processing
**bind_digit_action**: 
- Uses `switch_ivr_dmachine_*` for digit sequence matching
- Supports regex patterns, exact matches, timeout handling
- Realm-based binding for context isolation
- Action types: `exec` (app execution), `api` (API command), `transfer`

#### Media Manipulation  
**tone_detect**:
- DSP-based frequency detection
- Configurable frequency range, duration thresholds
- Progress tone detection (dial tone, busy, etc.)
- Callback execution on detection

#### File System Operations
**mkdir**: Directory creation with permission handling
**rename**: File/directory renaming with error checking

#### Session Management
**set_name**: Channel name modification for debugging/identification
**mutex**: Call flow synchronization via named mutex locks
**soft_hold**: Hold state without full hold treatment

**Loop**: `loop="N"` repeats action N times
**Substitution**: Allocated memory = `(strlen(data) + strlen(field_data) + 10) * proceed`

## Patterns
**Continue**: `continue=true` processes next extension after match
**Anti-actions**: Execute on condition failure
**Time**: `wday=1-7` (Sun=1), `hour=0-23`, use `tod_tz_offset` for timezone

## Gotchas
**Variable scope**: Hunt phase vars unavailable to same extension conditions - use `inline=true`
**Captures**: `$1-$10` condition-local only - save to channel vars with `inline=true`
**Memory**: Substitution buffer = `(strlen(data) + strlen(field) + 10) * matches`
**Time**: Returns -1 for no time attrs (absolute match), DST affects calculations, wday=1-7
**Anti-actions**: Not collected when `require-nested=true` and nested fail
**Extension names**: Used for `auto_hunt`, transfers, debug - unique per context

## Debug
**Logging**: `console loglevel debug`, timestamps with `SCF_DIALPLAN_TIMESTAMPS`
**Performance**: Order specific conditions first, use `break=on-true`, minimize inline actions, anchor regex patterns
**Errors**: Unclosed tags, missing data attrs, invalid break values, var scope misuse, capture scope misuse