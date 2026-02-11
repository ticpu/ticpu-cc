# mod_oreka - Call Recording Integration

Captures FreeSWITCH RTP streams, encodes to Opus, transmits to Oreka server via UDP SIP-like protocol. Supports dual/muxed streams, custom headers, per-channel config.

## Key Data Structures and Types

### Session Structure (`oreka_session_t`)
**RTP streams**: Separate read/write ports and streams
**Opus encoders**: Per-stream encoding (read/write)
**Media capture**: Media bug attachment, packet counters
**SIP integration**: Custom headers for INVITE/BYE
**Configuration**: Per-channel server addressing, mux option
```

### Global Configuration
**Network**: Local/server IPs, SIP port, UDP socket
**Opus settings**: Bitrate, complexity (0-10)
**Options**: Stream muxing, debug frame logging
```

### Enumerations
**Status**: `FS_OREKA_START`, `FS_OREKA_STOP`
**Stream types**: `FS_OREKA_READ` (RX), `FS_OREKA_WRITE` (TX)

## Core Functions

### `oreka_audio_callback()`
**Media bug handler**: INIT (Opus setup, SIP INVITE), CLOSE (SIP BYE, cleanup), READ/WRITE events
**Processing flow**: Linear PCM extraction → Opus encoding → RTP transmission → packet counting

### `encode_opus_frame()`
**Conversion**: 16-bit linear PCM → Opus with error logging

### `oreka_init_encoder()`
**Opus config**: VOIP application, voice signal, configurable bitrate/complexity, FEC disabled

## RTP Handling

### `oreka_setup_rtp()`
**Setup**: Port allocation → stream creation → Opus payload 96 → RFC 7587 (48kHz) → linear timestamps

### `oreka_tear_down_rtp()`
**Cleanup**: Port release → stream destruction → port reset → debug logging

## SIP Protocol

### `oreka_send_sip_message()`
**INVITE structure**: Standard SIP headers, custom Subject ("BEGIN RX/TX recording"), SDP with Opus payload 96
**SDP content**: FreeSWITCH originator, Oreka server connection, RTP ports, Opus parameters
**BYE message**: Same headers, no SDP

### Custom Headers
**Processing**: `oreka_sip_h_*` channel variables → SIP headers (prefix stripped)
**Storage**: Separate INVITE/BYE headers, original variables cleared

## Configuration (oreka.conf)

**Required**: `sip-server-addr`, `sip-server-port`
**Opus**: `opus-bitrate`, `opus-complexity` (0-10)
**Options**: `mux-all-streams`, `debug-frames`

### Channel Variables
**Override**: `oreka_sip_server_ipv4`, `oreka_sip_server_port`, `oreka_mux_streams`
**Headers**: `oreka_sip_h_*` (custom SIP headers)

### Application Interface
**`oreka_record [stop]`**: Start/stop recording, prevents duplicates, uses media bug framework

## FreeSWITCH Integration

### Media Bug Framework
**Muxed mode**: `SMBF_READ_STREAM | SMBF_WRITE_STREAM | SMBF_READ_PING | SMBF_ANSWER_REQ`
**Separate mode**: `SMBF_READ_REPLACE | SMBF_WRITE_REPLACE | SMBF_ANSWER_REQ`
**Capture points**: Read (remote), write (local), ping (muxed alternative)

### Session Lifecycle
**Flow**: App invocation → session allocation → media bug attachment → SIP INVITE → real-time processing → SIP BYE → cleanup

### Memory Management
**Pool allocation**, reference counting, automatic cleanup

## Debug Features

**Log levels**: ERROR (socket/RTP/Opus failures), WARNING (non-fatal issues), INFO (start/stop), DEBUG (streams/SIP/ports), DEBUG2 (headers)
**Frame debugging**: Configurable via `debug-frames` parameter


### Debug Points
**RTP**: Port allocation, stream creation
**Audio**: Frame counts, encoding status
**SIP**: Message content, headers
**Session**: Start/stop events, reference counting

## Network Communication

**UDP socket**: Single socket, matches server family, closed at shutdown
**SIP protocol**: Minimal SIP 2.0, standard headers, custom Subject, dynamic Content-Length
**RTP transmission**: Opus payload 96, linear timestamps, 48kHz clock, codec-derived packet time

## Error Handling

**Scenarios**: Missing config (load failure), network issues (logged), port exhaustion (graceful), Opus errors (logged), duplicates (prevented)
**Recovery**: Automatic RTP cleanup, proper Opus destruction, reference counting, socket error isolation

## Performance

**Memory**: Minimal footprint (~1KB Opus encoders, ~200B session)
**CPU**: VoIP-optimized Opus, configurable complexity, real-time processing, non-blocking
**Network**: Configurable bitrate (8-32kbps), minimal SIP overhead, standard RTP, UDP (no retransmission)

## Related Components

**Dependencies**: `switch.h`, `opus/opus.h`, `oreka.conf`
**Integration**: Media bug framework, RTP subsystem, socket API, XML config, application framework
**External**: Oreka server, Opus library, UDP network transport