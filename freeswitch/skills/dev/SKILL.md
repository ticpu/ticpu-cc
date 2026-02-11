---
name: dev
description: >
  ALWAYS load this skill before answering any FreeSWITCH C development
  question — writing/reviewing modules, codec implementations, media
  framework internals, transcoding, or CI/CD. Covers core media framework
  (switch_core_media), codec negotiation internals (Normal/Greedy/Scrooge),
  Opus FEC/PLC, AMR-WB OA/BE mode-set negotiation, Oreka recording,
  transcoding bridge IO path, and Cauca CI/CD pipeline. Also load the
  usage skill for dialplan/config/ESL, or the sofia skill for mod_sofia
  SIP internals.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Task
---

# FreeSWITCH Development Assistant

You are helping with FreeSWITCH internal C development: core media framework,
codec implementations, module APIs, and build infrastructure.

## Source Layout

```
src/
├── switch_core.c              Main core init/shutdown
├── switch_core_session.c      Session creation, lifecycle, threading
├── switch_core_state_machine.c Channel state machine driver
├── switch_core_io.c           Frame read/write (codec encode/decode path)
├── switch_core_media.c        SDP, codec negotiation, RTP session mgmt
├── switch_core_codec.c        Codec registry, init/destroy
├── switch_core_media_bug.c    Media bugs (record, eavesdrop, ASR hooks)
├── switch_rtp.c               RTP/RTCP (~9,100 lines)
├── switch_channel.c           Channel variables, flags, state transitions
├── switch_event.c             Event creation, fire, serialization
├── switch_ivr_bridge.c        Call bridging
├── switch_loadable_module.c   Module loader
├── include/
│   ├── switch.h               Master include
│   ├── switch_types.h         All enums, typedefs, constants
│   ├── switch_core.h          Core API declarations
│   └── switch_module_interfaces.h Module interface structs
└── mod/
    ├── endpoints/             SIP (mod_sofia), WebRTC (mod_verto), etc.
    ├── applications/          bridge, park, record, playback, conference
    ├── codecs/                G.711, G.729, Opus, AMR, etc.
    └── ...
```

## Module API Patterns

```c
SWITCH_MODULE_LOAD_FUNCTION(mod_example_load)
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_example_shutdown)
SWITCH_MODULE_RUNTIME_FUNCTION(mod_example_runtime)
SWITCH_MODULE_DEFINITION(mod_example, mod_example_load, mod_example_shutdown, mod_example_runtime);

#define SWITCH_STANDARD_API(name) static switch_status_t name(const char *cmd, switch_core_session_t *session, switch_stream_handle_t *stream)
#define SWITCH_STANDARD_APP(name) static void name(switch_core_session_t *session, const char *data)
```

## Core Conventions

- **APR memory pools**: Session owns a pool; pool destruction = automatic cleanup
- Allocate from session pool (`switch_core_session_alloc`) or module pool, never raw malloc for long-lived objects
- **`switch_status_t`**: Return type for nearly all API functions
- String: `switch_safe_strdup`, `zstr()` for null/empty checks
- Logging: `switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), ...)`
- Locking: `switch_mutex_lock`/`switch_thread_rwlock_rdlock` — always match lock/unlock
- Channel variables: `switch_channel_set_variable`, `_nodup` variant for pool-allocated strings

## Build System

- Autotools: `./bootstrap.sh && ./configure && make`
- Module selection: `modules.conf` (build-time), `modules.conf.xml` (load-time)
- License: MPL 1.1

## Detailed Reference

Read the reference docs bundled with this skill (same directory as this file).
Pick the relevant ones based on $ARGUMENTS before answering questions.

- [media_core_framework.md](media_core_framework.md) - Core media subsystem (switch_core_media)
- [media_transcoding_bridge_internals.md](media_transcoding_bridge_internals.md) - Transcoding bridge IO path
- [freeswitch_codec_negotiation_comprehensive.md](freeswitch_codec_negotiation_comprehensive.md) - Codec negotiation deep dive (Normal/Greedy/Scrooge)
- [codec_opus_implementation.md](codec_opus_implementation.md) - Opus FEC/PLC, bitrate, bandwidth
- [codec_amrwb_implementation.md](codec_amrwb_implementation.md) - AMR-WB OA/BE mode-set negotiation
- [codec_oreka_recording.md](codec_oreka_recording.md) - Oreka recording integration
- [freeswitch_cauca_cicd.md](freeswitch_cauca_cicd.md) - Cauca CI/CD infrastructure

## Guidelines

- Consult the reference docs first, verify with source code when needed
- Back answers with file:line evidence
- `switch_types.h` is the single source of truth for all enums and constants
- Trust source code over community wiki (may be outdated)
- For dialplan/config/ESL → use the `freeswitch` skill
- For mod_sofia SIP internals → use the `sofia` skill
