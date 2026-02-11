# FreeSWITCH Documentation Master Guide

Master guide to FreeSWITCH's SIP implementation, media framework, codecs, and dialplan. Maps technical information for debugging and implementation.

**Documentation Stats**: 1,891+ lines of essential content (60% reduction from original)

## Documentation Map

### SIP Implementation
- **[sip_mod_sofia_architecture.md](./sip_mod_sofia_architecture.md)** (269 lines) - Architecture, data structures, flags, debugging commands
- **[sip_mod_sofia_implementation.md](./sip_mod_sofia_implementation.md)** (400 lines) - Hard-coded values, SIP specifics, state machines, authentication, NAT algorithms
- **[sip_libsofia_integration.md](./sip_libsofia_integration.md)** (103 lines) - Sofia-SIP boundaries, NUA patterns, configuration, threading
- **[sip_media_negotiation.md](./sip_media_negotiation.md)** (222 lines) - SDP processing, codec negotiation, re-INVITE handling, hold/unhold

### Media Framework
- **[media_rtp_implementation.md](./media_rtp_implementation.md)** (570 lines) - RTP implementation, SRTP, DTMF, jitter buffers, statistics
- **[media_core_framework.md](./media_core_framework.md)** (192 lines) - Core media framework, SDP handling, ICE, security

### Codecs & Recording
- **[codec_opus_implementation.md](./codec_opus_implementation.md)** (100 lines) - Opus codec specifics, FEC, PLC, bitrate control
- **[codec_amrwb_implementation.md](./codec_amrwb_implementation.md)** (143 lines) - AMR-WB codec, payload formats, VoLTE compatibility
- **[codec_oreka_recording.md](./codec_oreka_recording.md)** (114 lines) - Recording integration, RTP forwarding

### Dialplan
- **[dialplan_xml_reference.md](./dialplan_xml_reference.md)** (300+ lines) - XML dialplan implementation, applications, conditions

## Quick Reference

### Key Constants
```c
#define SQL_CACHE_TIMEOUT 300              // 5 minutes
#define SOFIA_NAT_SESSION_TIMEOUT 90       // NAT timeout
#define MAX_CODEC_CHECK_FRAMES 50          // Codec validation
#define SOFIA_QUEUE_SIZE 50000             // Event queue size
```

### Critical Flags
- `PFLAG_AUTH_CALLS` `PFLAG_AGGRESSIVE_NAT_DETECTION` `PFLAG_3PCC` `PFLAG_AUTO_NAT`
- `TFLAG_RTP` `TFLAG_EARLY_MEDIA` `TFLAG_SIP_HOLD` `TFLAG_LATE_NEGOTIATION` `TFLAG_NAT`

### RFC Standards
- **RFC 3261** - SIP base specification
- **RFC 3264** - SDP Offer/Answer Model
- **RFC 7989** - Session-ID Header Field  
- **RFC 8760** - Multi-algorithm digest authentication

## Issue Location Guide

| Issue Type | Primary Document | Key Sections |
|------------|------------------|--------------|
| **Authentication failures** | sip_mod_sofia_implementation.md | Multi-algorithm auth, state machines |
| **SIP response issues** | sip_mod_sofia_implementation.md | Response code patterns, call flow |
| **Codec negotiation** | sip_media_negotiation.md | Negotiation modes, SDP processing |
| **Hold/resume problems** | sip_media_negotiation.md | Hold/unhold implementation |
| **No audio issues** | media_rtp_implementation.md | RTP activation, media streams |
| **DTMF problems** | media_rtp_implementation.md | RFC2833 handling |
| **NAT detection** | sip_mod_sofia_implementation.md | NAT algorithms, contact generation |
| **Registration issues** | sip_mod_sofia_implementation.md | State machines, gateway handling |
| **Performance issues** | sip_mod_sofia_implementation.md | Queue management, call limits |
| **Dialplan problems** | dialplan_xml_reference.md | Applications, conditions, execution flow |

## Troubleshooting Workflows

### No Audio Issues
1. Check RTP activation: sip_media_negotiation.md → Media Stream Setup
2. Verify codec negotiation: sip_media_negotiation.md → Codec Details  
3. Examine NAT handling: sip_mod_sofia_implementation.md → NAT Detection
4. Check RTP flags: media_rtp_implementation.md → Session management

### Authentication Failures  
1. Multi-algorithm support: sip_mod_sofia_implementation.md → RFC 8760
2. Nonce handling: sip_mod_sofia_implementation.md → Auth State Machine
3. Profile flags: sip_mod_sofia_implementation.md → PFLAGS

### Registration Problems
1. Gateway states: sip_mod_sofia_implementation.md → Registration States
2. Transport issues: sip_libsofia_integration.md → Transport Layer
3. NAT registration: sip_mod_sofia_implementation.md → NAT Handling

### Call Setup Failures
1. SIP responses: sip_mod_sofia_implementation.md → Response Patterns
2. Media negotiation: sip_media_negotiation.md → SDP Processing
3. Event processing: sip_libsofia_integration.md → Event Patterns

### Dialplan Issues
1. Application behavior: dialplan_xml_reference.md → Applications
2. Condition matching: dialplan_xml_reference.md → Conditions
3. Execution flow: dialplan_xml_reference.md → Architecture

## Configuration Impact

| Setting | Document | Effect |
|---------|----------|--------|
| `auth-calls` | sip_mod_sofia_implementation.md | Sets PFLAG_AUTH_CALLS |
| `aggressive-nat-detection` | sip_mod_sofia_implementation.md | Enhanced NAT algorithms |
| `codec-prefs` | sip_media_negotiation.md | Codec selection order |
| `enable-soa` | sip_media_negotiation.md | SOA vs manual SDP |
| `session-timeout` | sip_mod_sofia_implementation.md | SIP session timers |

## Advanced Topics

### Multi-Algorithm Authentication (RFC 8760)
- Implementation: sip_mod_sofia_implementation.md → Multi-Algorithm Auth
- Up to 7 simultaneous algorithm challenges
- Challenge generation and response validation

### Session-ID Implementation (RFC 7989)  
- Implementation: sip_mod_sofia_implementation.md → Session ID
- End-to-end session tracking across network elements

### Complex Media Scenarios
- Late negotiation: sip_media_negotiation.md → Proxy Mode
- Re-INVITE handling: sip_media_negotiation.md → Re-INVITE Changes
- Security: media_rtp_implementation.md + sip_media_negotiation.md → SRTP

### libsofia-sip Components

**NUA (Nokia User Agent)**: High-level SIP operations, call semantics
**NTA (Network Transaction API)**: SIP transaction layer, retransmissions
**SOA (SDP Offer/Answer)**: RFC 3264 implementation
**TPORT (Transport)**: Multi-transport support (UDP/TCP/TLS/WebSocket)

## Critical Magic Numbers

### Timeouts
- `SOFIA_NAT_SESSION_TIMEOUT 90` - NAT session timeout
- `DEFAULT_NONCE_TTL 60` - Authentication nonce lifetime
- `SQL_CACHE_TIMEOUT 300` - Database cache timeout

### Limits  
- `MAX_CODEC_CHECK_FRAMES 50` - Codec validation frames
- `MAX_MISMATCH_FRAMES 5` - Allowable codec mismatches
- `SOFIA_QUEUE_SIZE 50000` - Event queue capacity
- `SOFIA_MAX_MSG_QUEUE 64` - Message queue workers

### Performance Controls
- `SOFIA_MSG_QUEUE_SIZE 1000` - Messages per queue
- `ROUTE_MAX_HEADERS 20` - Route headers processed
- `SOFIA_MAX_REG_ALGS 7` - RFC8760 max algorithms

This optimized documentation provides essential technical references for FreeSWITCH experts debugging complex SIP interoperability issues, media problems, dialplan logic, and implementation-specific behavior.