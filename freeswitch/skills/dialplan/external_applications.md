# External Applications

File-like protocols used with playback, read, and other media applications.

## tone_stream://

Generates tones using Direct Digital Synthesis.

### Syntax

```
tone_stream://[settings;]%(duration_ms,wait_ms,freq1[,freq2,...])
```

- `duration_ms` - tone duration in milliseconds
- `wait_ms` - silence after tone in milliseconds
- `freq1,freq2,...` - frequencies in Hz (up to 18, mixed together)

DTMF characters `0-9*#A-D` play predefined dual-tone frequencies.

### Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `v=N` | Volume in dB (-63 to 0) | -7 |
| `d=N` | Default duration (ms) | 2000 |
| `w=N` | Default wait (ms) | 500 |
| `r=N` | Sample rate (Hz) | 8000 |
| `l=N` | Loops for next tone | 1 |
| `L=N` | Loops for entire script | 1 |
| `>=N` | Decay down over N ms | - |
| `<=N` | Decay up over N ms | - |

Append `;loops=N` to URI for repetition (`-1` for infinite).

### Examples

```xml
<!-- US ringback: 2s on, 4s off -->
<action application="playback" data="tone_stream://%(2000,4000,440,480)"/>

<!-- Busy signal (infinite) -->
<action application="playback" data="tone_stream://%(500,500,480,620);loops=-1"/>

<!-- UK ring (double ring pattern) -->
<action application="playback" data="tone_stream://%(400,200,400,450);%(400,2000,400,450)"/>

<!-- Conference beep -->
<action application="playback" data="tone_stream://%(200,0,500,600,700)"/>

<!-- Single 440Hz tone for 1 second -->
<action application="playback" data="tone_stream://%(1000,0,440)"/>

<!-- With settings: -10dB, 500ms duration -->
<action application="playback" data="tone_stream://v=-10;d=500;%(0,0,440)"/>
```
