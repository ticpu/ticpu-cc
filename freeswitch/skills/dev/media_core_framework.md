# FreeSWITCH Core Media

Central media coordinator: SDP negotiation, codec selection, RTP/SRTP/DTLS setup, ICE/STUN, T.38 fax, video handling.

## Key Data Structures and Types

### Core Media Handle
**Key elements:**
- **Engine array**: Separate RTP engines for audio/video/text
- **Multi-level mutexes**: Per-media type read/write, plus global SDP/control
- **Codec arrays**: Order preference, implementations, negotiated results, FMTP
- **Payload mappings**: IANA codes for media, DTMF, comfort noise
- **Crypto suite order**: Preference ranking for SRTP negotiation
```

### RTP Engine Structure
**Critical elements:**
- **Security arrays**: `ssec[CRYPTO_INVALID+1]` for all crypto types
- **Codec mismatch tracking**: Frame check counter, mismatch limits
- **Address variants**: Local, advertised, proxy IPs for NAT scenarios
- **ICE separation**: Incoming/outgoing ICE data structures
- **Video control flags**: FIR, PLI, NACK, TMMBR for quality management
- **T.140 text support**: Payload types and text factory
```



### SRTP Crypto Suites
**Suite hierarchy**: AEAD_AES_256_GCM preferred, AES_CM_128_HMAC_SHA1_80 common, key lengths 28-46 bytes
```

### Text Media Factory
**T.140 real-time text**: RED redundancy with `MAX_RED_FRAMES` buffers, timestamp tracking
```

## Core Functions

### `switch_core_media_negotiate_sdp()`
Main SDP negotiation: Sofia-SIP parsing, codec matching, ICE candidates, SRTP/DTLS setup

### T.38 Processing
- `switch_core_media_process_udptl()`: Extracts T.38 fax parameters from SDP
- `switch_core_media_extract_t38_options()`: Processes UDPTL media sections

### Media Flow Mapping
`sdp_media_flow()`: Maps SDP direction (sendonly/recvonly/sendrecv/inactive) to internal flow types

## Codec Negotiation

### Core Process
1. `switch_core_media_prepare_codecs()`: Parses preferences, loads implementations, handles FMTP, assigns payload types
2. `switch_core_media_set_codec()`: Validates selection, sets up read/write implementations, links with RTP
3. `switch_core_media_set_video_codec()`: Video-specific codec setup

### Negotiation Flow
**Local prep** → **Remote SDP parse** → **Codec matching** (preference order, payload conflicts) → **Activation** (RTP mapping, pipeline setup)

### Payload Management
- `switch_core_media_add_payload_map()`: Maps payload types to codecs, handles local/remote assignments, FMTP associations
- `switch_core_session_get_payload_code()`: Retrieves payload codes by IANA name, rate, FMTP parameters

## Media Stream Management

**Media handle coordination**: RTP engines per media type (audio/video/text), multi-level mutex protection

### Flow Control
- `switch_core_media_set_smode()`: Local send direction
- `switch_core_media_set_rmode()`: Remote send direction
- **Flow states**: SENDRECV, SENDONLY, RECVONLY, INACTIVE

### RTP Session Management
- `switch_core_media_set_rtp_session()`: Associates RTP session with engine
- `switch_core_media_get_rtp_session()`: Retrieves session by media type

### Auto-Adjustment
`switch_core_media_check_autoadj()`: Detects IP/port changes, NAT scenarios, validates consistent packet flow, respects `disable_rtp_auto_adjust` override

## Key Channel Variables

**Codec control**: `absolute_codec_string` (overrides all), `codec_string` (preference order)
**Media behavior**: `disable_rtp_auto_adjust`, `video_possible`, `has_t38`
**Debug/compatibility**: `NDLB_broken_opus_sdp`, crypto key storage


## Flags & Bug Handling

**Handle flags**: `SMF_INIT`, `SMF_READY`, `SMF_JB_PAUSED`, `SMF_VB_PAUSED`
**RTP bug workarounds**: `switch_core_media_parse_rtp_bugs()` handles timestamp, sequence, payload, NAT issues

### Video Functions
- `switch_core_media_get_video_fps()`: Calculates FPS, updates `video_fps` variable
- `switch_core_media_get_vid_params()`: Gets resolution, frame rate, bitrate for codec config

### Constants
**Quality limits**: `MAX_CODEC_CHECK_FRAMES=50`, `MAX_MISMATCH_FRAMES=5`
**Timing**: `VIDEO_REFRESH_FREQ=1000000us`, `TEXT_TIMER_MS=100`
**Buffers**: `MAX_RED_FRAMES=25`, `MAX_REJ_STREAMS=10`

## ICE Implementation


### `gen_ice()` Process
1. **MSID generation**: 32-byte random string for media stream ID
2. **CNAME generation**: 16-byte canonical name
3. **ICE credentials**: 16-byte ufrag, 24-byte password
4. **Candidate setup**: Foundation (10-digit), priority calculation, UDP transport

### ICE Resolution
`switch_core_media_set_resolveice()`: Controls hostname resolution in candidates, useful for NAT debugging

## Video Handling

### Video Thread Management
`switch_core_session_wake_video_thread()`: Validates handle, uses mutex + condition variable for thread wake

### Video Codec Functions
- `switch_core_media_set_video_codec()`: Configures codec with force option
- `switch_core_media_check_video_codecs()`: Validates configurations


### Video Global State
**CPU management**: `cpu_count`, `cur_cpu` for processing assignment
**Timing tracking**: Keyframe time, codec refresh, refresh requests
**Constants**: `VIDEO_REFRESH_FREQ=1000000us`

## T.140 Text Media

**Factory structure**: RED redundancy with buffer arrays, timestamps, position tracking
**Constants**: 100ms timer, 3s timeout, 25 max RED frames

### T.140 Features
**Protocol support**: Real-time text, RED redundancy (RFC 2198)
**Engine integration**: T.140 + RED payload types, factory instance
**Timing**: Precise transmission control, frame timestamp tracking

## Security (SRTP/DTLS)

**Crypto suite preference**: AEAD_AES_256_GCM → AES_CM_128_HMAC_SHA1_80 (common) → NULL_AUTH

### Crypto Functions
- `crypto_str2type()`/`crypto_type2str()`: String/type conversion
- `crypto_keysalt_len()`/`crypto_salt_len()`: Suite-specific length queries

### DTLS Support
**Per-engine fingerprints**: Local/remote certificates, controller flag
**Reinvite handling**: `check_dtls_reinvite()` manages role (client/server), version selection, RTP/RTCP setup
**Type configuration**: RTP always, RTCP if muxed

### Security Modes
**Requirements**: OPTIONAL (preferred), MANDATORY (required), FORBIDDEN
**Per-engine settings**: Array for all crypto types, active type, mode flags

## Debug Features

**Logging patterns**: Session-specific vs system-level, error contexts
**Key categories**: Codec negotiation (ADD PMAP), crypto keys, media parameters
**Debug levels**: ERROR, DEBUG, DEBUG1 for different verbosity


### Timeout & Quality Monitoring
**Timeout settings**: Max missed packets (normal/hold), media/hold timeouts
**Quality limits**: 50 check frames, 5 mismatch frames before action

**DEPRECATED profile params** (sofia.c:5409-5421, log WARNING on use):
- `rtp-timeout-sec` → replaced by channel variable `media_timeout`
- `rtp-hold-timeout-sec` → replaced by channel variable `media_hold_timeout`

**WARNING: unit change** (switch_core_media.c:2761):
The deprecated profile params are in **seconds**. The replacement channel
variables are in **milliseconds**. A direct value copy will break behavior
(`300` seconds ≠ `300` milliseconds).

**Scope change**: profile param → channel variable. The old params were
per-profile (all calls on that profile). The new variables are per-channel
(set in dialplan or per-call), allowing per-call tuning.

**Channel variables** (switch_core_media.c:2726, per-engine):
- `media_timeout` — ms of no RTP before hangup
- `media_hold_timeout` — ms of no RTP while on hold before hangup
- `media_timeout_audio` / `media_timeout_video` — per-media-type overrides
- `media_hold_timeout_audio` / `media_hold_timeout_video` — per-media-type hold overrides


## RTP Integration

### `switch_core_media_activate_rtp()`
**Activation flow**: Reinvite detection → media flag parsing → security flag setting → DTLS version setup


### Integration Points
**Session context**: Media handle ↔ session linkage, channel variables, multi-level mutexes
**Module interfaces**: Sofia-SIP, codec modules, file I/O endpoints

### Media Pipeline
**Read path**: RTP reception → codec decoding → frame delivery → jitter buffer
**Write path**: Frame reception → codec encoding → RTP transmission → timing control

### Thread Management
**Media helper**: Condition variables, file protection mutexes, ready state flags
**Thread types**: Per-engine processing, dedicated video write threads

## T.38 Fax Processing

**Key functions**: `extract_t38_options()` (SDP parsing), `process_udptl()` (parameter extraction), `process_t38_passthru()` (endpoint bridging)
**Features**: SDP UDPTL parsing, transmission parameters, passthrough mode, `has_t38` flag

## Key Related Files

**Core**: `switch_rtp.c` (RTP/SRTP/DTLS), `switch_stun.c` (ICE/NAT), `switch_core_codec.c` (codec management)
**SIP integration**: `sofia-sip/` library, `mod_sofia/` endpoint
**Config**: `vars.xml` (codec preferences), `switch.conf.xml` (RTP settings)
**Debug**: `sofia loglevel all 9`, `uuid_media debug <uuid>`, SIP traces