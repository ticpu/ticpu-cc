# LibSofia-SIP FreeSWITCH Integration Specifics

## Key Integration Points

**Primary Callback**: `sofia_event_callback()` - all SIP events from libsofia-sip funnel through this single function

**Handle Binding**: NUA handles created via `nua_handle()` bind to FreeSWITCH `sofia_private_t` objects

**State Isolation**: FreeSWITCH call states operate independently of SIP dialog states - requires explicit coordination

**SOA Integration**: `SOATAG_*` tags pass SDP data between layers, bypassing direct SDP string manipulation

## Configuration Specifics

**NUA Creation**: 
```c
profile->nua = nua_create(profile->s_root, sofia_event_callback, profile,
    NUTAG_URL(profile->bindurl),
    NTATAG_USER_VIA(1),      /* Force User Via */
    SOATAG_AF(...),          /* Address family */
    TAG_END());
```

**Runtime Control**: `nua_set_params()` with FreeSWITCH-specific tags:
```c
TPTAG_LOG(debug_level)        /* Per-profile debug */
TPTAG_CAPT(capture_server)    /* HEP capture */
NUTAG_AUTOANSWER(auto_answer) /* Profile auto-answer */
```

**Event Callback Signature**:
```c
void sofia_event_callback(nua_event_t event, int status, char const *phrase,
    nua_t *nua, sofia_profile_t *profile,
    nua_handle_t *nh, sofia_private_t *sofia_private,
    sip_t const *sip, tagi_t tags[])
```

## Integration Patterns

**Handle Creation**:
```c
tech_pvt->nh = nua_handle(profile->nua, sofia_private,
    SIPTAG_FROM_STR(from_str),
    SIPTAG_TO_STR(to_str), TAG_END());
```
**Mapping**: Each SIP dialog = 1 NUA handle + 1 `sofia_private_t` + Session UUID

**Event Processing**:
```c
switch (event) {
case nua_i_invite: process_invite(profile, nh, sip, sofia_private); break;
case nua_r_invite: process_invite_response(profile, nh, sip, sofia_private); break;
case nua_i_bye: process_bye(profile, nh, sip, sofia_private); break;
}
```

**SDP Operations**:
```c
nua_invite(tech_pvt->nh,
    SOATAG_USER_SDP_STR(local_sdp),
    SOATAG_AUDIO_AUX("cn telephone-event"), /* FreeSWITCH aux codecs */
    TAG_END());

nua_respond(nh, 200, "OK", SOATAG_USER_SDP_STR(answer_sdp), TAG_END());
```

## Threading Model

**Sofia Constraints**: Single event loop (`su_root_t`) - NOT thread-safe
**FreeSWITCH Pattern**: Each profile = dedicated thread + Sofia event loop
**Cross-Thread**: SIP events queued for worker threads when needed

## Memory Management

**Sofia**: Home-based allocation (`su_home_t`) + reference counting
**FreeSWITCH**: Session pools + profile pools with clear ownership boundaries

## Extensions

**Custom Headers**: `SIPTAG_HEADER_STR("X-FreeSWITCH-Var: value")`
**Custom Methods**: `NUTAG_APPL_METHOD("CUSTOM_METHOD")`
**Transport Extensions**: WebSocket for WebRTC, custom TPTAG_* parameters

## Debug Levels

- `TPORT_LOG`: Transport layer
- `NTA_DEBUG`: Transaction layer  
- `NUA_DEBUG`: User agent layer
- `SOFIA_DEBUG`: General Sofia

**FreeSWITCH Integration**: Per-profile control, runtime modification, integrated logging

## Performance

**Bottlenecks**: Event queue latency, memory allocation churn, DNS blocking, TLS handshake overhead
**Optimizations**: Tuned queues, connection pooling, DNS caching, async processing

## Critical Issues

**NAT Traversal**: Sofia handles basic Via processing, FreeSWITCH manages advanced NAT policy + STUN + media proxy decisions
**Timer Conflicts**: Coordination required between Sofia SIP timers and FreeSWITCH session timers
**Auth Loops**: FreeSWITCH implements circuit breakers for infinite challenge loops
**Media Sync**: State machine coordination prevents SIP/RTP mismatches