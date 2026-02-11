# Sofia Media Negotiation Specifics

## Critical Functions

**Main Entry**: `sofia_media_negotiate_sdp()` - delegates to `switch_core_media_negotiate_sdp()`
**RTP Setup**: `sofia_media_activate_rtp()` - thread-safe wrapper around core activation
**Payload Tracking**: Special payloads for telephone events (`te`, `recv_te`, `bte`) and comfort noise (`cng_pt`, `bcng_pt`)

## Key Structure Elements
```c
struct private_object {
    switch_core_media_params_t mparams;  // Core media params
    uint8_t flags[TFLAG_MAX];  // Sofia-specific flags
    switch_payload_t te, recv_te, bte;  // DTMF event payloads
    switch_payload_t cng_pt, bcng_pt;  // Comfort noise payloads
    switch_rtp_bug_flag_t rtp_bugs;  // Vendor workarounds
    uint32_t req_media_counter;  // Media request tracking
};
```

## SDP Processing

**Main Function**:
```c
uint8_t sofia_media_negotiate_sdp(switch_core_session_t *session, 
    const char *r_sdp, switch_sdp_type_t type) {
    uint8_t t, p = 0;
    private_object_t *tech_pvt = switch_core_session_get_private(session);
    
    if ((t = switch_core_media_negotiate_sdp(session, r_sdp, &p, type))) {
    sofia_set_flag_locked(tech_pvt, TFLAG_SDP);
    }
    if (!p) sofia_set_flag(tech_pvt, TFLAG_NOREPLY);  // No processing needed
    return t;
}
```

**Sequence**: Port selection → SDP generation → RTP activation
**Constraints**: Duplicate SDP detection, type handling (OFFER/ANSWER/REINVITE), transport protocol support

## Codec Negotiation

**Profile → Session Transfer**:
```c
tech_pvt->mparams.inbound_codec_string = profile->inbound_codec_string;
tech_pvt->mparams.outbound_codec_string = profile->outbound_codec_string;
```

**Negotiation Modes**:
- Normal: Standard matching
- Greedy (`SCMF_CODEC_GREEDY`): Accept any supported
- Scrooge (`SCMF_CODEC_SCROOGE`): Most restrictive

**Payload Types**: Static (0-95), Dynamic (96-127), Special (DTMF: `te`/`recv_te`/`bte`, CNG: `cng_pt`/`bcng_pt`)
**fmtp Processing**: Sample rate, channels, codec-specific params (OPUS complexity, G.729 annexes)

## RTP Activation

**Thread-Safe Wrapper**:
```c
switch_status_t sofia_media_activate_rtp(private_object_t *tech_pvt) {
    switch_mutex_lock(tech_pvt->sofia_mutex);
    status = switch_core_media_activate_rtp(tech_pvt->session);
    switch_mutex_unlock(tech_pvt->sofia_mutex);
    
    if (status == SWITCH_STATUS_SUCCESS) {
    sofia_set_flag(tech_pvt, TFLAG_RTP | TFLAG_IO);
    }
    return status;
}
```

**SRTP**: Profile `require-secure-rtp`, RTP fallback, `SM_NDLB_DISABLE_SRTP_AUTH` Asterisk workaround
**Directions**: sendrecv (default), sendonly (remote hold), recvonly (local hold), inactive (mutual hold)
**Streams**: Audio (primary), Video, Text (T.140), Application (T.38 fax, BFCP)

## Re-INVITE Processing

**Hold Detection**:
```c
if (switch_stristr("sendonly", r_sdp) || 
    switch_stristr("0.0.0.0", r_sdp) || 
    switch_stristr("inactive", r_sdp)) {
    hold_related = 1;
}
```

**Processing Flow**:
```c
match = sofia_media_negotiate_sdp(session, r_sdp, SDP_OFFER);
if (match) {
    switch_core_media_choose_port(session, SWITCH_MEDIA_TYPE_AUDIO, 0);
    switch_core_media_gen_local_sdp(session, SDP_ANSWER, NULL, 0, NULL, 0);
    if (sofia_media_activate_rtp(tech_pvt) != SWITCH_STATUS_SUCCESS) {
    switch_channel_hangup(channel, SWITCH_CAUSE_DESTINATION_OUT_OF_ORDER);
    }
} else {
    nua_respond(tech_pvt->nh, SIP_488_NOT_ACCEPTABLE, TAG_END());
}
```

## Hold/Unhold

**Detection**: `sendonly` attribute, `0.0.0.0` connection, `inactive` attribute
**Flags**: `TFLAG_SIP_HOLD` (Sofia), `TFLAG_SIP_HOLD_INACTIVE` (inactive type), `CF_LEG_HOLDING` (channel)
**Proxy Hold**: `PFLAG_PROXY_HOLD` - local hold/unhold without bridge impact

## Early Media

**Setup**:
```c
case nua_callstate_early:
    sofia_set_flag_locked(tech_pvt, TFLAG_EARLY_MEDIA);
    switch_channel_mark_pre_answered(channel);
    match = sofia_media_negotiate_sdp(session, r_sdp, SDP_ANSWER);
    if (match) {
    switch_core_media_choose_port(session, SWITCH_MEDIA_TYPE_AUDIO, 0);
    if (sofia_media_activate_rtp(tech_pvt) != SWITCH_STATUS_SUCCESS) {
    switch_channel_hangup(channel, SWITCH_CAUSE_DESTINATION_OUT_OF_ORDER);
    }
    }
```

**183 Handling**:
```c
if (status == 183 && !r_sdp) {
    if (sofia_test_pflag(profile, PFLAG_IGNORE_183NOSDP)) goto done;
    status = 180;  // Treat as 180 Ringing
}
if (status == 180 && r_sdp) status = 183;  // Convert 180+SDP to 183
```

## Media Modes

**Types**: Normal (full processing), Proxy (`CF_PROXY_MODE` - SIP only), Proxy Media (`CF_PROXY_MEDIA` - RTP proxy), Bypass (direct RTP), Late Negotiation (`TFLAG_LATE_NEGOTIATION` - deferred)

**Selection Logic**:
```c
if (switch_channel_test_flag(channel, CF_PROXY_MODE)) {
    switch_channel_set_variable(channel, SWITCH_ENDPOINT_DISPOSITION_VARIABLE, "RECEIVED_NOMEDIA");
} else if (switch_channel_test_flag(channel, CF_PROXY_MEDIA)) {
    switch_channel_set_variable(channel, SWITCH_ENDPOINT_DISPOSITION_VARIABLE, "PROXY MEDIA");
    switch_core_media_patch_sdp(session);
} else if (sofia_test_flag(tech_pvt, TFLAG_LATE_NEGOTIATION)) {
    switch_channel_set_variable(channel, SWITCH_ENDPOINT_DISPOSITION_VARIABLE, "DELAYED NEGOTIATION");
} else {
    if (sofia_media_tech_media(tech_pvt, r_sdp, SDP_OFFER) != SWITCH_STATUS_SUCCESS) {
    nua_respond(nh, SIP_488_NOT_ACCEPTABLE, TAG_END());
    switch_channel_hangup(channel, SWITCH_CAUSE_INCOMPATIBLE_DESTINATION);
    }
}
```

## Core Media Integration

**Key Functions**: `switch_core_media_negotiate_sdp()`, `switch_core_media_choose_port()`, `switch_core_media_activate_rtp()`, `switch_core_media_gen_local_sdp()`
**Handle Lifecycle**: Session creation → profile config → cleanup
**Coordination**: Core handles ports, Sofia syncs flags

## SOA Framework

**Modes**: SOA (`TFLAG_ENABLE_SOA` - Nokia library) vs Manual (direct construction)

**SOA Response**:
```c
if (sofia_use_soa(tech_pvt)) {
    nua_respond(tech_pvt->nh, SIP_200_OK,
    SOATAG_USER_SDP_STR(tech_pvt->mparams.local_sdp_str),
    SOATAG_REUSE_REJECTED(1),
    SOATAG_AUDIO_AUX("cn telephone-event"), TAG_END());
} else {
    nua_respond(tech_pvt->nh, SIP_200_OK,
    NUTAG_MEDIA_ENABLE(0),
    SIPTAG_CONTENT_TYPE_STR("application/sdp"),
    SIPTAG_PAYLOAD_STR(tech_pvt->mparams.local_sdp_str), TAG_END());
}
```

**Key Tags**: `SOATAG_USER_SDP_STR()`, `SOATAG_REUSE_REJECTED(1)`, `SOATAG_AUDIO_AUX()`

## Advanced Features

**ICE**: Profile-level config, core integration, NAT traversal, STUN/TURN
**T.38**: SDP `m=image` detection, re-INVITE processing, UDPTL activation
**Multi-Stream**: Audio (primary), video, text (accessibility), application
**Transcoding**: Codec compatibility, profile settings, resources, routing

## Error Handling

**Failures**:
- Codec negotiation: `488 Not Acceptable`, `SWITCH_CAUSE_INCOMPATIBLE_DESTINATION`
- RTP activation: "RTP Error" logs, `SWITCH_CAUSE_DESTINATION_OUT_OF_ORDER`
- Port selection: "Port Error!" logs, early return
- Re-INVITE: `488`/`491` responses, flag-based recovery

**Recovery**: Flag-based tracking, mutex protection, proxy fallbacks, clean hangup handling

## Performance

**Optimizations**: SDP caching, lazy codec prep, port pooling, profile memory pools
**Scalability**: Profile-shared config, bit-based flags, minimal memory, connection reuse

## Critical Configuration

**Profile Settings**:
```xml
<param name="codec-prefs" value="PCMU,PCMA,G729,G722,OPUS"/>
<param name="inbound-codec-negotiation" value="generous|greedy|scrooge"/>
<param name="media-option" value="resume-media-on-hold"/>
<param name="require-secure-rtp" value="true"/>
<param name="ignore-183nosdp" value="true"/>
<param name="enable-soa" value="true"/>
```

**Runtime Variables**: `absolute_codec_string`, `ep_codec_string`, `sip_ignore_reinvites`, `bypass_media_resume_on_hold`, `rtp_disable_hold`

## Debug Info

**Log Messages**: "Early Media RTP Error!", "Reinvite Codec Error!", "Codec Negotiation Error!", "Port Error!", "Duplicate SDP"

**Critical Flags**: `TFLAG_SDP`, `TFLAG_EARLY_MEDIA`, `TFLAG_RTP`, `TFLAG_REINVITED`, `TFLAG_SIP_HOLD`, `CF_REINVITE`, `CF_PROXY_MODE`

**Common Issues**: No audio (RTP/port), codec problems (config/negotiation), hold/resume (detection/flags), re-INVITE failures (SDP/matching), early media (183 handling)