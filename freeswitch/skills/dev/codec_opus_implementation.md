# mod_opus - OPUS Codec Implementation

RFC 7587 OPUS codec: 6-510 kbps, 8-48kHz, FEC/PLC, dynamic bitrate, VBR/CBR/DTX, asymmetric rates.

## Key Data Structures and Types

### Codec Settings
**FEC/DTX**: `useinbandfec`, `usedtx` flags
**Bitrate**: `maxaveragebitrate`, `cbr` mode
**Timing**: `maxptime`/`minptime`/`ptime` (10-120ms)
**Capability signaling**: `sprop_*` parameters
```

### Context Structure
**OPUS objects**: Encoder/decoder instances with frame sizes
**FEC features**: `use_jb_lookahead`, `look_check`, packet loss tracking
**Statistics**: Encoder/decoder stats, runtime control state
**Adaptation**: Decoder recreation flag, dynamic sample rate
```

### Statistics & Control
**Decoder stats**: FEC/PLC corrections, frame count
**Encoder stats**: Frames/bytes/time encoded, FEC usage
**Dynamic control**: Current/wanted bitrate, increase/decrease steps, FEC preservation

## Default Profiles
**48kHz**: 30kbps, 40ms max ptime, FEC+DTX
**16kHz**: 20kbps, 60ms max ptime
**8kHz**: 14kbps, 120ms max ptime

## Core Functions

### `switch_opus_encode()`
PCM → OPUS encoding with statistics, debug logging, FEC detection

### `switch_opus_decode()`
**OPUS → PCM decoding**:
- **PLC mode**: `SFF_PLC` flag triggers packet loss concealment
- **FEC recovery**: Jitter buffer lookahead finds FEC in future packets
- **Flexible timing**: Adapts to different packet sizes

### `switch_opus_encode_repacketize()`
**Multi-frame encoding**: For 80-120ms packets, smart FEC on last frame for next packet, uses `OpusRepacketizer`

## FMTP Parameters

**Supported**: `useinbandfec`, `usedtx`, `cbr`, `maxptime`/`minptime`/`ptime`, `stereo`/`sprop-stereo`, `maxaveragebitrate` (6-510kbps), rate parameters

### Bitrate Limits
**Range**: 6-510 kbps, FEC requires ≥12.4 kbps

### Bandwidth Mapping
**8kHz**: NARROWBAND, **12kHz**: MEDIUMBAND, **16kHz**: WIDEBAND, **24kHz**: SUPERWIDEBAND, **48kHz**: FULLBAND

## Initialization

`switch_opus_init()`: FMTP parsing → asymmetric rate negotiation → OPUS object creation → parameter config → RFC 7587 RTP setup

### Control Interface
**`switch_opus_control()`**:
- `SCC_DEBUG`: Debug level
- `SCC_AUDIO_PACKET_LOSS`: Encoder adjustment
- `SCC_AUDIO_ADJUST_BITRATE`: "increase"/"decrease"/"default" commands
- `SCC_AUDIO_VAD`: Voice activity detection
- `SCC_CODEC_SPECIFIC`: `jb_lookahead` etc.

### RTP Integration
**Fixed 48kHz RTP clock** (RFC 7587), `opus_ts_step()` timestamp calc, `switch_rtp_set_interval()` config

## Performance

**FEC**: Jitter buffer lookahead, bitrate thresholds, `switch_opus_keep_fec_enabled()` adjustment
**Memory**: Buffer reuse, careful pointer management
**CPU**: Debug-only FEC detection, efficient FMTP handling, direct OPUS API

## Debug Features

**Debug levels**: 0 (none), 1 (basic), 2+ (detailed)
**`switch_opus_info()`**: Frame/sample counts, bandwidth, FEC presence, validation
**Statistics**: Encoder/decoder stats, runtime logging
**API**: `opus_debug on/off`, version info

## Configuration (opus.conf.xml)

**Quality**: `use_vbr`, `use_dtx`, `complexity` (0-10)
**Bitrate**: `maxaveragebitrate`, `bitrate_negotiation`, `adjust_bitrate`
**FEC**: `keep_fec`, `fec_decode`, `use_jb_lookahead`
**Compatibility**: `asymmetric_samplerates`, `mono`
**Debug**: `debuginfo` level


## Error Handling

**Validation**: Sample rates (8-48kHz), bitrates (6-510kbps), FMTP bounds, buffer sizes
**Recovery**: Decoder recreation, FEC/PLC fallbacks, error statistics
**OPUS integration**: Error codes, `opus_strerror()`, version checking

## Media Framework Integration

**Registrations**: Multiple sample rates (8/16/48kHz), mono/stereo, packet times (10-120ms), IANA 116
**Features**: Audio handling, transcoding, channel bridging, format negotiation