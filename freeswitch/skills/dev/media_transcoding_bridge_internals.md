# FreeSWITCH Media Transcoding and Bridge Internals

This document details the internal workings of FreeSWITCH's audio transcoding pipeline, focusing on the bridge audio loop, codec handling, resampling, and the distinction between the "perfect" and "buffered" transcoding paths.

## Overview

When two call legs are bridged in FreeSWITCH, audio flows bidirectionally between them. If the codecs differ (e.g., Opus on one leg, PCMU on the other), FreeSWITCH must transcode audio in real-time. This involves:

1. **Decoding** the source codec to linear PCM (L16)
2. **Resampling** if sample rates differ (e.g., 8kHz ↔ 48kHz)
3. **Encoding** to the destination codec

## Bridge Audio Loop

**File:** `src/switch_ivr_bridge.c:799-827`

The bridge is remarkably simple - it reads frames from one session and writes them to the other:

```c
/* read audio from 1 channel and write it to the other */
status = switch_core_session_read_frame(session_a, &read_frame, SWITCH_IO_FLAG_NONE, stream_id);

if (SWITCH_READ_ACCEPTABLE(status)) {
    if (switch_test_flag(read_frame, SFF_CNG)) {
        // Handle comfort noise
        if (silence_val) {
            switch_generate_sln_silence((int16_t *) silence_frame.data, ...);
            read_frame = &silence_frame;
        } else if (!switch_channel_test_flag(chan_b, CF_ACCEPT_CNG)) {
            continue;
        }
    }

    // Write to the other leg - this is where transcoding happens
    if (switch_core_session_write_frame(session_b, read_frame, SWITCH_IO_FLAG_NONE, stream_id) != SWITCH_STATUS_SUCCESS) {
        goto end_of_bridge_loop;
    }
}
```

The complexity is hidden inside `switch_core_session_write_frame()`, which handles all transcoding.

## Write Frame Entry Point

**File:** `src/switch_core_media.c:15884-15961`

When a frame is written to a session, the system first determines if transcoding is needed:

```c
// Line 15898: Check if codecs differ
if (frame->codec && session->write_codec->implementation != frame->codec->implementation) {
    if (session->write_impl.codec_id == frame->codec->implementation->codec_id ||
        session->write_impl.microseconds_per_packet != frame->codec->implementation->microseconds_per_packet) {
        ptime_mismatch = TRUE;
    }
    need_codec = TRUE;
}

// Line 15925: Check if sample rates differ
if (frame->codec->implementation->actual_samples_per_second != session->write_impl.actual_samples_per_second) {
    need_codec = TRUE;
    do_resample = TRUE;
}

// Line 15958: No transcoding needed - fast path
if (!need_codec) {
    do_write = TRUE;
    write_frame = frame;
    goto done;
}
```

## Decoding Phase

**File:** `src/switch_core_media.c:15972-15994`

If transcoding is required, the source frame is first decoded to L16:

```c
if (frame->codec) {
    session->raw_write_frame.datalen = session->raw_write_frame.buflen;
    status = switch_core_codec_decode(frame->codec,           // Source codec (e.g., PCMU)
                                      session->write_codec,   // Target codec (e.g., Opus)
                                      frame->data,            // Encoded data
                                      frame->datalen,
                                      session->write_impl.actual_samples_per_second,  // Target rate
                                      session->raw_write_frame.data,    // Output: L16 PCM
                                      &session->raw_write_frame.datalen,
                                      &session->raw_write_frame.rate,
                                      &frame->flags);

    if (do_resample && status == SWITCH_STATUS_SUCCESS) {
        status = SWITCH_STATUS_RESAMPLE;
    }
}
```

**Example:** PCMU @ 8kHz, 20ms ptime
- Input: 160 bytes encoded
- Output: 320 bytes L16 (160 samples × 2 bytes/sample)

## Resampler Creation

**File:** `src/switch_core_media.c:15997-16020`

If the decoder returns `SWITCH_STATUS_RESAMPLE`, a resampler is created:

```c
case SWITCH_STATUS_RESAMPLE:
    resample++;
    write_frame = &session->raw_write_frame;
    write_frame->rate = frame->codec->implementation->actual_samples_per_second;

    if (!session->write_resampler) {
        switch_mutex_lock(session->resample_mutex);
        status = switch_resample_create(&session->write_resampler,
            frame->codec->implementation->actual_samples_per_second,  // From: 8000 Hz
            session->write_impl.actual_samples_per_second,            // To: 48000 Hz
            session->write_impl.decoded_bytes_per_packet,
            SWITCH_RESAMPLE_QUALITY,
            session->write_impl.number_of_channels);
        switch_mutex_unlock(session->resample_mutex);

        switch_log_printf(..., SWITCH_LOG_NOTICE, "Activating write resampler\n");
    }
    break;
```

## Resampling Phase

**File:** `src/switch_core_media.c:16079-16109`

Once the resampler exists, L16 data is converted to the target sample rate:

```c
if (session->write_resampler) {
    short *data = write_frame->data;

    switch_mutex_lock(session->resample_mutex);
    if (session->write_resampler) {
        // Resample the PCM data
        switch_resample_process(session->write_resampler, data,
            write_frame->datalen / 2 / session->write_resampler->channels);

        // Copy resampled data back
        memcpy(data, session->write_resampler->to,
            session->write_resampler->to_len * 2 * session->write_resampler->channels);

        // Update frame metadata
        write_frame->samples = session->write_resampler->to_len;
        write_frame->datalen = write_frame->samples * 2 * session->write_resampler->channels;
        write_frame->rate = session->write_resampler->to_rate;

        did_write_resample = 1;
    }
    switch_mutex_unlock(session->resample_mutex);
}
```

**Example:** PCMU → Opus resampling
- Input: 320 bytes L16 @ 8kHz (160 samples, 20ms)
- Output: 1920 bytes L16 @ 48kHz (960 samples, 20ms)

## Perfect vs Buffered Transcoding Paths

After decoding and resampling, the system chooses between two encoding paths based on whether packet sizes match.

### Perfect Path Decision

**File:** `src/switch_core_media.c:16181-16185`

```c
if (session->write_codec) {
    if (!ptime_mismatch && write_frame->codec && write_frame->codec->implementation &&
        write_frame->codec->implementation->decoded_bytes_per_packet == session->write_impl.decoded_bytes_per_packet) {
        perfect = TRUE;
    }
}
```

**Perfect path is chosen when:**
- No ptime mismatch
- Frame codec's decoded packet size matches session's write codec packet size

**Example where perfect path FAILS:**
- Frame codec: PCMU @ 8kHz → `decoded_bytes_per_packet` = 320 bytes
- Write codec: Opus @ 48kHz → `decoded_bytes_per_packet` = 1920 bytes
- 320 ≠ 1920 → `perfect = FALSE` → Takes buffered path

### Perfect Path (Lines 16189-16258)

When packet sizes match, encoding happens directly without buffering:

```c
if (perfect) {
    enc_frame = write_frame;
    session->enc_write_frame.datalen = session->enc_write_frame.buflen;

    status = switch_core_codec_encode(session->write_codec,
                                      frame->codec,
                                      enc_frame->data,
                                      enc_frame->datalen,
                                      session->write_impl.actual_samples_per_second,  // CORRECT: Uses target rate
                                      session->enc_write_frame.data,
                                      &session->enc_write_frame.datalen,
                                      &session->enc_write_frame.rate,
                                      &flag);

    // ... handle encode result ...

    status = perform_write(session, write_frame, flags, stream_id);
    goto error;  // Exit (not actually an error, just early return)
}
```

**Key point:** Line 16206 correctly uses `session->write_impl.actual_samples_per_second` for the encoder's sample rate parameter.

### Buffered Path (Lines 16260-16430)

When packet sizes don't match, data is accumulated in a buffer before encoding:

```c
} else {
    // Create buffer if needed
    if (!session->raw_write_buffer) {
        switch_size_t bytes_per_packet = session->write_impl.decoded_bytes_per_packet;
        switch_log_printf(..., SWITCH_LOG_DEBUG,
            "Engaging Write Buffer at %u bytes to accommodate %u->%u\n",
            (uint32_t) bytes_per_packet, write_frame->datalen,
            session->write_impl.decoded_bytes_per_packet);

        status = switch_buffer_create_dynamic(&session->raw_write_buffer,
            bytes_per_packet * SWITCH_BUFFER_BLOCK_FRAMES,
            bytes_per_packet * SWITCH_BUFFER_START_FRAMES, 0);
    }

    // Write resampled data to buffer
    switch_buffer_write(session->raw_write_buffer, write_frame->data, write_frame->datalen);
```

Then, when enough data is buffered, encoding proceeds:

```c
    // Encode when we have enough data
    while (switch_buffer_inuse(session->raw_write_buffer) >= session->write_impl.decoded_bytes_per_packet) {
        int rate;

        // Read from buffer
        session->raw_write_frame.datalen = switch_buffer_read(session->raw_write_buffer,
            session->raw_write_frame.data,
            session->write_impl.decoded_bytes_per_packet);

        enc_frame = &session->raw_write_frame;
        session->raw_write_frame.rate = session->write_impl.actual_samples_per_second;

        // Determine rate for encoder - THIS IS WHERE THE BUG IS
        if (frame->codec && frame->codec->implementation && switch_core_codec_ready(frame->codec)) {
            rate = frame->codec->implementation->actual_samples_per_second;  // BUG: Uses source rate!
        } else {
            rate = session->write_impl.actual_samples_per_second;  // Correct fallback
        }

        // Encode
        status = switch_core_codec_encode(session->write_codec,
                                          frame->codec,
                                          enc_frame->data,
                                          enc_frame->datalen,
                                          rate,  // Passed to encoder
                                          session->enc_write_frame.data,
                                          &session->enc_write_frame.datalen,
                                          &session->enc_write_frame.rate,
                                          &flag);
        // ...
    }
}
```

## Sample Rate Bug in Buffered Path

**File:** `src/switch_core_media.c:16302-16306`

The buffered path has a critical bug at lines 16302-16306:

```c
if (frame->codec && frame->codec->implementation && switch_core_codec_ready(frame->codec)) {
    rate = frame->codec->implementation->actual_samples_per_second;
} else {
    rate = session->write_impl.actual_samples_per_second;
}
```

**Problem:** This uses `frame->codec` (the source codec, e.g., PCMU @ 8kHz) to determine the rate, but the data in `enc_frame` has already been resampled to the target rate (e.g., 48kHz).

**Comparison with Perfect Path:**

| Path | Rate Source | Value | Correct? |
|------|-------------|-------|----------|
| Perfect (line 16206) | `session->write_impl.actual_samples_per_second` | 48000 | ✓ |
| Buffered (line 16303) | `frame->codec->implementation->actual_samples_per_second` | 8000 | ✗ |

## BUG Codec System

**File:** `src/switch_core_io.c:448-460`

FreeSWITCH maintains a separate "bug codec" for media bug processing (recording, eavesdrop, etc.):

```c
// Create bug_codec if needed
if (!switch_core_codec_ready(&session->bug_codec) && switch_core_codec_ready(read_frame->codec)) {
    switch_core_codec_copy(read_frame->codec, &session->bug_codec, NULL, NULL);
    switch_log_printf(..., SWITCH_LOG_DEBUG, "Setting BUG Codec %s:%d\n",
        session->bug_codec.implementation->iananame,
        session->bug_codec.implementation->ianacode);
}
```

During codec changes, the bug codec is destroyed and lazily recreated:

**File:** `src/switch_core_codec.c:174-180`

```c
/* force media bugs to copy the read codec from the next frame */
switch_thread_rwlock_wrlock(session->bug_rwlock);
if (switch_core_codec_ready(&session->bug_codec)) {
    switch_log_printf(..., "Destroying BUG Codec %s:%d\n", ...);
    switch_core_codec_destroy(&session->bug_codec);
}
switch_thread_rwlock_unlock(session->bug_rwlock);
```

## Session Reset During Codec Change

**File:** `src/switch_core_session.c:1409-1422`

When a codec changes mid-call, `switch_core_session_reset()` destroys resamplers and buffers:

```c
/* clear resamplers */
switch_mutex_lock(session->resample_mutex);
switch_resample_destroy(&session->read_resampler);
switch_resample_destroy(&session->write_resampler);
switch_mutex_unlock(session->resample_mutex);

/* ... */

switch_buffer_destroy(&session->raw_write_buffer);
switch_buffer_destroy(&session->raw_read_buffer);
```

Resamplers and buffers are recreated lazily on the next frame that needs them.

## Codec Change Sequence

**File:** `src/switch_core_media.c:3636-3750`

The full codec change sequence during a re-INVITE:

1. `switch_core_session_reset()` - Destroys resamplers and buffers
2. `switch_channel_audio_sync()` - Synchronizes audio
3. `switch_yield()` - Deliberate 20ms pause
4. Destroy old read/write codecs
5. Initialize new read codec
6. Initialize new write codec
7. `switch_rtp_reset_jb()` - Reset jitter buffer (discards packets!)
8. `switch_core_session_set_real_read_codec()` - Sets new read codec, destroys bug_codec

## Frame Size Reference

| Codec | Sample Rate | Ptime | Samples/Frame | L16 Bytes |
|-------|-------------|-------|---------------|-----------|
| PCMU | 8000 Hz | 20ms | 160 | 320 |
| PCMA | 8000 Hz | 20ms | 160 | 320 |
| G722 | 16000 Hz | 20ms | 320 | 640 |
| Opus | 48000 Hz | 20ms | 960 | 1920 |

## Key Data Structures

```c
// Session-level codec state
session->read_codec      // Current read codec
session->write_codec     // Current write codec
session->read_impl       // Read codec implementation details
session->write_impl      // Write codec implementation details
session->bug_codec       // Codec for media bug processing

// Resampler state
session->read_resampler  // For resampling read frames
session->write_resampler // For resampling write frames

// Frame buffers
session->raw_read_frame  // Decoded read frame
session->raw_write_frame // Decoded write frame (for buffered path)
session->enc_write_frame // Encoded output frame

// Buffered transcoding
session->raw_write_buffer // Buffer for accumulating decoded samples
```

## Related Files

| File | Purpose |
|------|---------|
| `src/switch_core_media.c` | Main media/codec handling |
| `src/switch_core_io.c` | Frame read/write, bug codec |
| `src/switch_core_codec.c` | Codec lifecycle management |
| `src/switch_core_session.c` | Session state, reset |
| `src/switch_ivr_bridge.c` | Bridge audio loop |
| `src/switch_rtp.c` | RTP packet handling |
| `src/switch_resample.c` | Audio resampling |
