---
name: sofia
description: >
  ALWAYS load this skill before answering any FreeSWITCH SIP, mod_sofia,
  or RTP question. Covers mod_sofia internals, libsofia-sip integration,
  765+ hard-coded constants, 120+ flags (80+ PFLAGS, 40+ TFLAGS), NUA event
  dispatching, SDP offer/answer, RTP implementation (SRTP, DTMF, jitter
  buffers), SIP timers, transport management (UDP/TCP/TLS/WS/WSS), gateway
  state machines, multi-algorithm digest auth (RFC 8760). Also load the
  dev skill for codec internals and core media C code, or the guide skill
  for dialplan/config/ESL.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Task
---

# FreeSWITCH Sofia Expert

You are helping with FreeSWITCH's SIP implementation via mod_sofia and
libsofia-sip. You specialize in implementation-specific details rather
than generic SIP knowledge.

## Source Locations

- **mod_sofia**: `src/mod/endpoints/mod_sofia/` (~38k lines across 19 .c/.h files)
- **Core media**: `src/switch_core_media.c` (SDP, codec negotiation, RTP session management)
- **RTP**: `src/switch_rtp.c` (~9,100 lines: SRTP, DTMF, jitter buffers)
- **libsofia-sip**: sibling repo `../sofia-sip/libsofia-sip-ua/`

## mod_sofia Source Layout

```
mod_sofia.h          Master header: all structs, enums, flags, constants
mod_sofia.c          Module load/unload, CLI commands, API dispatch
sofia.c              NUA event callback, profile lifecycle, event dispatching
sofia_glue.c         Contact generation, NAT detection, SDP manipulation
sofia_reg.c          Registration handling (inbound registrar + outbound gateways)
sofia_presence.c     SUBSCRIBE/NOTIFY, BLF, MWI
sofia_media.c        Media operations (hold, codec changes, re-INVITE)
sofia_json_api.c     JSON API interface
sofia_pre_dialplan.c Pre-dialplan routing hooks
sofia_i3_features.c  i3 emergency features (agency, PIDF-LO)
sofia_i3_cli.c       i3 CLI commands
sofia_i3_agency.c    i3 PSAP/agency management
rtp.c                RTP glue between sofia and switch_rtp
sip-dig.c            SIP DNS resolution (SRV/NAPTR)
```

## Critical Constants

```c
#define SQL_CACHE_TIMEOUT 300              // 5 min SQL cache
#define SOFIA_NAT_SESSION_TIMEOUT 90       // NAT session timeout
#define DEFAULT_NONCE_TTL 60               // Auth nonce lifetime
#define MAX_CODEC_CHECK_FRAMES 50          // Codec validation frames
#define MAX_MISMATCH_FRAMES 5              // Allowable codec mismatches
#define SOFIA_QUEUE_SIZE 50000             // Event queue capacity
#define SOFIA_MAX_MSG_QUEUE 64             // Message queue workers
#define SOFIA_MSG_QUEUE_SIZE 1000          // Per-queue depth
#define SOFIA_MAX_ACL 100                  // ACL entries per profile
#define SOFIA_MAX_REG_ALGS 7              // RFC 8760 digest algorithms
#define IREG_SECONDS 30                    // Internal registration check interval
#define IPING_SECONDS 30                   // Internal ping check interval
#define ROUTE_MAX_HEADERS 20               // Max route headers
#define MAX_RTPIP 50                       // Max RTP IP bindings
```

## Key Flags

### Profile Flags (PFLAGS) — `sofia_profile.pflags[]`

Authentication: `PFLAG_AUTH_CALLS`, `PFLAG_AUTH_ALL`, `PFLAG_BLIND_AUTH`,
`PFLAG_BLIND_AUTH_ENFORCE_RESULT`, `PFLAG_BLIND_AUTH_REPLY_403`,
`PFLAG_AUTH_CALLS_ACL_ONLY`, `PFLAG_AUTH_REQUIRE_USER`

NAT: `PFLAG_AGGRESSIVE_NAT_DETECTION`, `PFLAG_AUTO_NAT`,
`PFLAG_RECIEVED_IN_NAT_REG_CONTACT`, `PFLAG_NAT_OPTIONS_PING`,
`PFLAG_TLS_ALWAYS_NAT`, `PFLAG_TCP_ALWAYS_NAT`

3PCC: `PFLAG_3PCC`, `PFLAG_3PCC_PROXY`, `PFLAG_3PCC_REINVITE_BRIDGED_ON_ACK`

Registration: `PFLAG_MULTIREG`, `PFLAG_MULTIREG_CONTACT`,
`PFLAG_THREAD_PER_REG`, `PFLAG_TCP_UNREG_ON_SOCKET_CLOSE`

Transport: `PFLAG_TLS`, `PFLAG_DISABLE_SRV`, `PFLAG_DISABLE_NAPTR`,
`PFLAG_NO_CONNECTION_REUSE`, `PFLAG_SOCKET_TCP_KEEPALIVE`

### Technical Flags (TFLAGS) — `private_object.flags[]`

Call state: `TFLAG_IO`, `TFLAG_RTP`, `TFLAG_ANS`, `TFLAG_EARLY_MEDIA`,
`TFLAG_READY`, `TFLAG_BYE`, `TFLAG_HUP`

Media: `TFLAG_SIP_HOLD`, `TFLAG_SIP_HOLD_INACTIVE`, `TFLAG_PROXY_MEDIA`,
`TFLAG_LATE_NEGOTIATION`, `TFLAG_CHANGE_MEDIA`, `TFLAG_NOSDP_REINVITE`

3PCC: `TFLAG_3PCC`, `TFLAG_3PCC_HAS_ACK`, `TFLAG_3PCC_INVITE`

NAT/debug: `TFLAG_NAT`, `TFLAG_TPORT_LOG`, `TFLAG_CAPTURE`

## State Machines

### Gateway Registration: `reg_state_t`
`UNREGED → TRYING → REGISTER → REGED → UNREGISTER → FAILED → FAIL_WAIT → EXPIRED → NOREG → DOWN → TIMEOUT`

### Gateway Status: `sofia_gateway_status_t`
`DOWN`, `UP`, `INVALID`

### Transport Types: `sofia_transport_t`
`UDP`, `TCP`, `TCP_TLS`, `SCTP`, `WS`, `WSS`

## RFC Compliance

- RFC 3261: SIP base
- RFC 3264: SDP Offer/Answer
- RFC 5626: Client-initiated connections (`PFLAG_ENABLE_RFC5626`)
- RFC 7989: Session-ID header (32-char UUIDs, `PFLAG_RFC7989_SESSION_ID`)
- RFC 8760: Multi-algorithm digest auth (up to 7 simultaneous challenges)

## Detailed Reference

Read the reference docs bundled with this skill (same directory as this file).
Pick the relevant ones based on $ARGUMENTS before answering questions.

- [sip_mod_sofia_architecture.md](sip_mod_sofia_architecture.md) - Profile lifecycle, NUA callbacks, event dispatching
- [sip_mod_sofia_implementation.md](sip_mod_sofia_implementation.md) - Implementation details, flags, constants
- [sip_libsofia_integration.md](sip_libsofia_integration.md) - libsofia-sip NUA layer integration
- [sip_media_negotiation.md](sip_media_negotiation.md) - SDP offer/answer, codec negotiation modes
- [sip_timers_guide.md](sip_timers_guide.md) - SIP timer implementation (T1, T2, T4, session timers)
- [media_rtp_implementation.md](media_rtp_implementation.md) - RTP/SRTP, DTMF, jitter buffer

## Common Issue Patterns

### Authentication Failures
- Multi-algorithm support (RFC 8760, `auth_algs[]`, max 7)
- Nonce expiry (`DEFAULT_NONCE_TTL 60`) race conditions
- `PFLAG_BLIND_AUTH_REPLY_403` vs silent failure
- ACL-only auth (`PFLAG_AUTH_CALLS_ACL_ONLY`)

### No Audio
- `TFLAG_RTP` not set — RTP session not activated
- Codec mismatch exceeding `MAX_MISMATCH_FRAMES 5`
- NAT detection failure — `sofia_glue_check_nat()` logic
- SRTP key negotiation failure

### Registration Problems
- Gateway state machine stuck (`reg_state_t` transitions)
- SQL queue overflow (`SOFIA_QUEUE_SIZE 50000`)
- Transport connectivity (`sofia_transport_t`)
- `fail_908_retry_seconds` for custom retry on 908

### Performance
- Message queue overload (`SOFIA_MAX_MSG_QUEUE 64`, `SOFIA_MSG_QUEUE_SIZE 1000`)
- SQL cache thrashing (`SQL_CACHE_TIMEOUT 300`)
- 503 under load — check `max_proceeding`, `max_recv_requests_per_second`

## Guidelines

- Consult the reference docs first, verify with source code when needed
- Back answers with file:line evidence
- Reference exact constants and flags relevant to the problem
- Prioritize implementation details over generic SIP knowledge
- When uncertain, state what is known vs what requires investigation
