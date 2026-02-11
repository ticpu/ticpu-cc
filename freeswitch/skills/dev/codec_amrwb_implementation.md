# mod_amrwb - AMR-WB Codec

RFC 4867 AMR-WB codec: 9 modes (6.6-23.85 kbps), 16kHz, Octet Aligned/Bandwidth Efficient formats, VoLTE support, dynamic bitrate.

## Compilation Modes
**AMRWB_PASSTHROUGH**: FMTP pass-through only, no codec operations
**Full mode**: OpenCore AMR-WB encoding/decoding, complete FMTP processing

## Key Data Structures and Types

### Global Configuration
**Bitrate control**: `default_bitrate`, `adjust_bitrate`, `mode_set_overwrite`
**Format**: `force_oa`, `volte` compatibility
**Debug**: Global debug flag

### Context Structure
**OpenCore states**: Encoder/decoder instances
**Mode control**: `enc_modes` bitmask, current `enc_mode`, change period
**Timing**: `max_ptime`, `ptime` parameters
**Flags**: Option flags, redundancy, per-session debug
```

### Option Flags
**Format**: `OCTET_ALIGN`, `CRC`, `ROBUST_SORTING`, `INTERLEAVING`
**Control**: `MODE_CHANGE_NEIGHBOR`

### Bitrate Modes (0-8)
**Low**: 6.6kbps (0), 8.85kbps (1), 12.65kbps (2)
**Mid**: 14.25kbps (3), 15.85kbps (4), 18.25kbps (5)
**High**: 19.85kbps (6), 23.05kbps (7), 23.85kbps (8)
```

## Core Functions

### `switch_amrwb_init()`
**Initialization**: Context allocation, FMTP parsing, OpenCore state init, outgoing FMTP generation
**FMTP processing**: `octet-align`, `mode-set`, `mode-change-*`, `crc`, `robust-sorting`, `interleaving`, timing params
**Mode selection**: Highest available mode, or global default, with overwrite option

### `switch_amrwb_encode()`
**Process**: Validation → OpenCore `E_IF_encode()` → CMR header (0xf0) → OA/BE packing → debug info

### `switch_amrwb_decode()`
**Process**: Size validation (max 62 bytes) → buffer copy → OA/BE unpacking → OpenCore `D_IF_decode()` → length setting

### `switch_amrwb_destroy()`
**Cleanup**: `E_IF_exit()`, `D_IF_exit()`, pointer clearing

## Configuration

### Frame Sizes
**Mode sizes**: 17,23,32,36,40,46,50,58,60 bytes (modes 0-8), plus SID frames

### Defaults
**Bitrate**: Mode 8 (23.85 kbps), **Max size**: 62 bytes
**Sample rate**: 16kHz, **Frame**: 320 samples (20ms), **Output**: 640 bytes

### VoLTE Mode
**FMTP additions**: `max-red=0;mode-change-capability=2` (ETSI TS 126 236 compliance)

## Payload Formats

### Octet Aligned (OA)
**Unpacking**: CMR extraction, TOC processing, frame validation, dynamic bitrate support
**Packing**: Placeholder implementation

### Bandwidth Efficient (BE)
**Processing**: External implementations, bit-level packing, `switch_amr_array_lshift()`

### Frame Validation
**Valid types**: Modes 0-8, SID frames, SPEECH_LOST (0xE), NO_DATA (0xF)

## FreeSWITCH Integration

### Codec Registration
**Two implementations**: OA (payload 100), BE (payload 110), shared functions, different FMTP defaults

### Implementation Parameters
**Specs**: 16kHz, 23.85kbps, 20ms (320 samples), 640 decoded bytes, mono
**Functions**: init/encode/decode/destroy
```

### Dynamic Bitrate
**When enabled**: `SWITCH_CODEC_FLAG_HAS_ADJ_BITRATE`, runtime changes via control interface

## Control Interface

### `switch_amrwb_control()`
**SCC_DEBUG**: Per-session debug level
**SCC_AUDIO_ADJUST_BITRATE**: "increase" (+2 modes), "decrease" (-2 modes), "default" (global), other (min)
**Step size**: 2 modes for noticeable quality change

## FMTP Handling

### Generation (`generate_fmtp`)
**Format**: `octet-align=<0|1>;mode-set=<mode>;[VoLTE]`
**VoLTE**: `max-red=0;mode-change-capability=2`
**Mode-set**: Comma-separated values for multiple modes

## Debug Features

### `switch_amrwb_info()`
**Analysis**: Frame type, quality indicator, multi-frame flag, payload/total sizes
**Output format**: "FT: [0x8] Q: [0x1] Frame flag: [0] payload sz: [60] len: [61]"

### API Commands
**`amrwb_debug <on|off>`**: Global debug toggle (non-passthrough mode only)

### Debug Hierarchy
**Global**: `globals.debug` (API/config), **Per-session**: `context->debug` (control), **Function-specific**: Temporary calls

## Configuration (amrwb.conf)

**Parameters**:
- `default-bitrate`: Mode 0-8
- `volte`: VoLTE FMTP params
- `adjust-bitrate`: Runtime adjustment
- `force-oa`: Force Octet Aligned
- `mode-set-overwrite`: Override negotiation

## Error Handling

**Validation**: Frame types, sizes (vs frame table), OpenCore errors, buffer overflow (62-byte limit)
**Key messages**: "Invalid TOC", "Invalid frame size", "E_IF_encode() ERROR", "passthrough mode only"

## RTP Integration

**Payload formats**: RFC 4867 OA/BE, CMR processing, multi-frame support
**Negotiation**: FMTP parsing, format selection, compatibility fallbacks
**Timing**: 20ms default, configurable ptime/maxptime, frame alignment

## Related Files

**Core**: `mod_amrwb.c/.h`, `amrwb_be.h`, `bitshift.h`
**OpenCore**: `E_IF_*`/`D_IF_*` functions, external library dependencies
**Integration**: Core codec framework, RTP payload, SDP/FMTP, media bridging
**Debug**: `amrwb.conf`, `amrwb_debug` console commands

## Technical Notes

**Memory**: Pool allocation, automatic cleanup, separate OpenCore state management
**Threading**: Per-codec context, read-only global config, OpenCore-dependent safety
**Performance**: CPU-intensive operations, bitrate/quality trade-offs, debug overhead
**Compatibility**: RFC 4867, ETSI TS 126 236 (VoLTE), OpenCore/API version dependencies