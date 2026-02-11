# FreeSWITCH Codec Negotiation: Complete Implementation Guide

## Overview

This document provides comprehensive guidance on FreeSWITCH codec negotiation, including maximum flexibility configurations, common misconceptions, and implementation details based on source code analysis.

## Table of Contents

1. [Codec Negotiation Modes](#codec-negotiation-modes)
2. [Late Negotiation and SDP-less Calls](#late-negotiation-and-sdp-less-calls)
3. [Opus Codec Configuration](#opus-codec-configuration)
4. [Codec String Formats and Ptime Handling](#codec-string-formats-and-ptime-handling)
5. [Liberal Acceptance Configuration](#liberal-acceptance-configuration)
6. [Implementation Details](#implementation-details)
7. [Common Misconceptions](#common-misconceptions)

## Codec Negotiation Modes

FreeSWITCH supports three distinct codec negotiation modes (found in `src/switch_core_media.c:4861-4874`):

### Generous Mode (Most Liberal)
```xml
<action application="set" data="rtp_codec_negotiation=generous"/>
```
- Disables both greedy and scrooge flags
- Provides maximum codec flexibility
- **Best choice for liberal acceptance**

### Greedy Mode
```xml
<action application="set" data="rtp_codec_negotiation=greedy"/>
```
- More aggressive in codec selection
- Prioritizes FreeSWITCH's preferences over remote

### Scrooge Mode (Most Restrictive)
```xml
<action application="set" data="rtp_codec_negotiation=scrooge"/>
```
- Most restrictive mode
- Enables both greedy and scrooge flags

## Late Negotiation and SDP-less Calls

### Configuration
```xml
<!-- Profile-level setting -->
<param name="inbound-late-negotiation" value="true"/>

<!-- Channel variable -->
<action application="set" data="sip_enable_late_negotiation=true"/>
```

### Important Implementation Details

**Key Finding**: `inbound-late-negotiation` is profile-level but **outbound calls explicitly disable late negotiation** in the code (`mod_sofia.c:3364`):
```c
sofia_clear_flag_locked(tech_pvt, TFLAG_LATE_NEGOTIATION);
```

**Exceptions**: 3PCC scenarios with bypass media can re-enable late negotiation for outbound calls.

**Conclusion**: Late negotiation primarily affects **inbound calls only**, despite configuration name suggesting broader scope.

## Opus Codec Configuration

### Basic Configuration (`/conf/autoload_configs/opus.conf.xml`)
```xml
<configuration name="opus.conf">
  <settings>
    <!-- Enable FEC for maximum compatibility -->
    <param name="advertise-useinbandfec" value="true"/>

    <!-- Enable DTX for bandwidth efficiency -->
    <param name="use-dtx" value="true"/>

    <!-- Use VBR for quality -->
    <param name="use-vbr" value="true"/>

    <!-- Accept asymmetric sample rates -->
    <param name="asymmetric-sample-rates" value="true"/>

    <!-- Liberal bitrate range -->
    <param name="maxaveragebitrate" value="128000"/>
  </settings>
</configuration>
```

### Sample Rate Parameters

#### maxplaybackrate
- **`maxplaybackrate=48000`**: FreeSWITCH can decode up to 48kHz quality
- **`maxplaybackrate=0`**: **Removes the parameter entirely from SDP FMTP** (no explicit playback rate limit)
- **Unconfigured**: Uses preset defaults (48000/16000/8000 depending on preset)

**Implementation detail**: From `mod_opus.c:582`:
```c
if (settings->maxplaybackrate) {
    snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), "maxplaybackrate=%d; ", settings->maxplaybackrate);
}
```

#### sprop-maxcapturerate
- **`sprop-maxcapturerate=48000`**: FreeSWITCH will encode up to 48kHz
- **`sprop-maxcapturerate=0`**: **Removes the parameter entirely from SDP FMTP** (no explicit capture rate signaling)
- **Unconfigured**: Uses preset defaults (0 for normal, 8000/16000 for others)

**SDP Impact**: When set to 0, these parameters are completely omitted from the `a=fmtp` line in SDP offers/answers, giving maximum negotiation flexibility by not constraining the remote endpoint.

### Asymmetric Sample Rate Calculation
With `asymmetric_samplerates=true`:
- **Encoder**: `min(remote_maxplaybackrate, local_sprop_maxcapturerate)`
- **Decoder**: `min(local_maxplaybackrate, remote_sprop_maxcapturerate)`

## Codec String Formats and Ptime Handling

### Critical Implementation Detail

**Source**: `src/switch_loadable_module.c:2858-2861`

```c
// Key decision logic in codec selection:
if ((!interval && (uint32_t) (imp->microseconds_per_packet / 1000) != default_ptime) ||
    (interval && (uint32_t) (imp->microseconds_per_packet / 1000) != interval)) {
    continue;  // Skip this implementation
}
```

### Ptime Behavior Analysis

| Format | Behavior | Flexibility |
|--------|----------|-------------|
| `OPUS` | Defaults to exactly 20ms only | Restrictive to 20ms |
| `OPUS@20i` | Explicitly 20ms only | Restrictive to 20ms |
| `OPUS@10i,OPUS@20i,OPUS@40i` | Multiple ptime support | Maximum flexibility |

### Common Misconception
**WRONG**: `OPUS` (without ptime) is more flexible than `OPUS@20i`
**CORRECT**: Both `OPUS` and `OPUS@20i` are equally restrictive to 20ms only

### Default Ptime Function
```c
SWITCH_DECLARE(uint32_t) switch_default_ptime(const char *name, uint32_t number)
{
    uint32_t *p;

    if ((p = switch_core_hash_find(runtime.ptimes, name))) {
        return *p;
    }

    return 20;  // Default fallback is 20ms
}
```

### Real-World SDP Impact
If remote sends `a=ptime:40` in SDP:
- ❌ `OPUS` will **fail to match** (restricted to 20ms)
- ❌ `OPUS@20i` will **fail to match** (restricted to 20ms)
- ✅ `OPUS@40i` will **successfully match**

## Liberal Acceptance Configuration

### Complete Sofia Profile Configuration
```xml
<profile name="liberal">
  <!-- Liberal codec preferences with multiple ptimes -->
  <param name="codec-prefs" value="OPUS@10i,OPUS@20i,OPUS@40i,OPUS@60i,G722,PCMU,PCMA,G729,GSM"/>
  <param name="inbound-codec-prefs" value="OPUS@10i,OPUS@20i,OPUS@40i,OPUS@60i,G722,PCMU,PCMA,G729,GSM"/>
  <param name="outbound-codec-prefs" value="OPUS@10i,OPUS@20i,OPUS@40i,OPUS@60i,G722,PCMU,PCMA,G729,GSM"/>

  <!-- Enable late negotiation for inbound -->
  <param name="inbound-late-negotiation" value="true"/>

  <!-- Disable codec restrictions -->
  <param name="disable-transcoding" value="false"/>
</profile>
```

### Maximum Flexibility Dialplan
```xml
<extension name="liberal_outbound">
  <condition field="destination_number" expression="^(.*)$">
    <!-- Use generous codec negotiation -->
    <action application="set" data="rtp_codec_negotiation=generous"/>

    <!-- Prefer remote SDP codec choices -->
    <action application="set" data="ep_codec_prefer_sdp=true"/>

    <!-- Liberal codec preferences with multiple ptimes -->
    <action application="set" data="absolute_codec_string=OPUS@10i,OPUS@20i,OPUS@40i,OPUS@60i,G722@20i,PCMU@20i,PCMA@20i"/>

    <!-- Bridge the call -->
    <action application="bridge" data="sofia/gateway/your_gateway/$1"/>
  </condition>
</extension>
```

### ep_codec_prefer_sdp Variable
```xml
<action application="set" data="ep_codec_prefer_sdp=true"/>
```

**Function**: Changes codec iteration from local-preference-driven to remote-SDP-driven

#### Implementation Details
**Source**: `src/switch_core_media.c:13665`
```c
if (switch_channel_direction(channel) == SWITCH_CALL_DIRECTION_INBOUND || prefer_sdp) {
    // Use SDP-first iteration order
}
```

#### How It Works
1. **Without `ep_codec_prefer_sdp`** (default for outbound):
   - FreeSWITCH iterates through **local codec preferences first**
   - Tries to match each local codec against remote SDP
   - Results in codec order: `[local_pref1, local_pref2, ...]`

2. **With `ep_codec_prefer_sdp=true`**:
   - FreeSWITCH iterates through **remote SDP codecs first**
   - Tries to match each remote codec against local preferences
   - Results in codec order: `[remote_sdp1, remote_sdp2, ...]`

#### Practical Example
**Local preferences**: `OPUS@20i,G722,PCMU`
**Remote SDP**:
```
m=audio 5004 RTP/AVP 0 8 97
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 OPUS/48000/2
```

**Without `ep_codec_prefer_sdp=true`**:
- Final codec string: `OPUS,PCMU` (local order preserved)

**With `ep_codec_prefer_sdp=true`**:
- Final codec string: `PCMU,OPUS` (remote SDP order preserved)

#### Key Behaviors
- **Automatic for inbound calls**: Always uses SDP-first iteration
- **Manual for outbound calls**: Must explicitly set `ep_codec_prefer_sdp=true`
- **Integration**: Works with generous/greedy/scrooge modes for acceptance criteria
- **Result storage**: Creates `ep_codec_string` channel variable
- **Use cases**: SIP trunk compatibility, interoperability with systems expecting their codec order

## Implementation Details

### Key Source Files
- **Codec Selection**: `src/switch_loadable_module.c:2746-2794`
- **Negotiation Logic**: `src/switch_core_media.c:4861-4874`
- **SDP Processing**: `src/switch_core_media.c:13442-13466` (`switch_core_media_merge_sdp_codec_string()`)
- **Codec Iteration Control**: `src/switch_core_media.c:13534-13665` (`switch_core_media_set_r_sdp_codec_string()`)
- **Sofia Implementation**: `src/mod/endpoints/mod_sofia/sofia_media.c:37`
- **Opus Module**: `src/mod/codecs/mod_opus/mod_opus.c`

### Opus Codec Registrations
The Opus module registers implementations for:
- **Sample Rates**: 48kHz, 16kHz, 8kHz
- **Ptimes**: 10ms, 20ms, 40ms, 60ms, 80ms, 100ms, 120ms
- **Channels**: Mono and stereo for each combination

### Function Call Chain and Dependencies

#### Core Negotiation Flow
1. **`switch_core_media_merge_sdp_codec_string()`** (`src/switch_core_media.c:13442`)
   - Entry point for SDP processing
   - Calls `switch_core_media_set_r_sdp_codec_string()` at line 13466
   - Passes session and SDP information

2. **`switch_core_media_set_r_sdp_codec_string()`** (`src/switch_core_media.c:13534`)
   - Reads `ep_codec_prefer_sdp` channel variable (line 13562)
   - Sets `prefer_sdp` flag based on variable value
   - Controls codec iteration order (line 13665)
   - **Critical Early Exit**: Returns immediately if channel is NULL or no codecs loaded

3. **Codec Iteration Logic** (`src/switch_core_media.c:13665`)
   ```c
   if (switch_channel_direction(channel) == SWITCH_CALL_DIRECTION_INBOUND || prefer_sdp) {
       // Remote SDP first iteration
   } else {
       // Local preference first iteration
   }
   ```

#### Prerequisites for ep_codec_prefer_sdp
- **Valid Channel**: `switch_core_session_get_channel(session)` must return non-NULL
- **Loaded Codecs**: `num_codecs > 0` from codec loading functions
- **Proper Session**: Session structure must be properly initialized
- **Variable Access**: Channel variable system must be functional

### Constants
- `MAX_CODEC_CHECK_FRAMES = 50`: Maximum frames for codec validation
- `MAX_MISMATCH_FRAMES = 5`: Allowable codec mismatches before rejection

## Common Misconceptions

### Misconception 1: Simple vs Detailed Codec Strings
**WRONG**: `OPUS@20i` is more restrictive than `OPUS`
**CORRECT**: Both are equally restrictive to 20ms only

### Misconception 2: Default Ptime Flexibility
**WRONG**: Omitting ptime gives more negotiation flexibility
**CORRECT**: Omitting ptime still restricts to default (20ms) only

### Misconception 3: inbound-late-negotiation Scope
**WRONG**: `inbound-late-negotiation` affects both inbound and outbound calls
**CORRECT**: Primarily affects inbound calls; outbound calls explicitly disable it

### Misconception 4: Liberal Acceptance with Simple Formats
**WRONG**: Using `OPUS` accepts any ptime the remote offers
**CORRECT**: Must explicitly list multiple ptime variants for true flexibility

### Misconception 5: ep_codec_prefer_sdp Always Works
**WRONG**: Setting `ep_codec_prefer_sdp=true` always changes codec iteration order
**CORRECT**: Requires proper session/channel initialization and loaded codecs to function

#### Common Failure Scenarios
- **NULL Channel**: If `switch_core_session_get_channel()` returns NULL
- **No Codecs Loaded**: If codec loading fails or returns zero codecs
- **Early Function Exit**: `switch_core_media_set_r_sdp_codec_string()` exits before reading the variable
- **Incomplete Call Chain**: Missing calls to `switch_core_media_merge_sdp_codec_string()`

## Best Practices

### For Maximum Codec Flexibility
1. **Use generous negotiation mode**
2. **Enable SDP preference** with `ep_codec_prefer_sdp=true`
3. **Explicitly list multiple ptime variants** instead of relying on defaults
4. **Configure liberal Opus settings** with asymmetric rates
5. **Enable late negotiation** for inbound calls

### For Reliable ep_codec_prefer_sdp Operation
1. **Ensure Proper Session Initialization**: Session and channel structures must be valid
2. **Verify Codec Loading**: Confirm codecs are properly loaded before SDP processing
3. **Call Function Chain**: Ensure `switch_core_media_merge_sdp_codec_string()` is called
4. **Monitor Debug Output**: Check for early exits in `switch_core_media_set_r_sdp_codec_string()`
5. **Validate Channel Variables**: Confirm channel variable system is functional

### Example Complete Configuration
```xml
<!-- Profile settings -->
<param name="inbound-late-negotiation" value="true"/>
<param name="codec-prefs" value="OPUS@10i,OPUS@20i,OPUS@40i,OPUS@60i,G722,PCMU,PCMA"/>

<!-- Dialplan variables -->
<action application="set" data="rtp_codec_negotiation=generous"/>
<action application="set" data="ep_codec_prefer_sdp=true"/>
<action application="set" data="absolute_codec_string=OPUS@10i,OPUS@20i,OPUS@40i,OPUS@60i,G722@20i,PCMU@20i,PCMA@20i"/>
```

This configuration maximizes FreeSWITCH's ability to accept any codec it can transcode while leveraging its robust transcoding capabilities for format conversion.