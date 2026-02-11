# FreeSWITCH mod_sofia Deep Implementation Analysis

## Hard-coded Values and Magic Numbers

### Core Constants
```c
#define SQL_CACHE_TIMEOUT 300  // 5 minutes SQL cache
#define DEFAULT_NONCE_TTL 60  // 60 seconds nonce TTL for auth
#define IREG_SECONDS 30  // Internal reg check interval
#define IPING_SECONDS 30  // Internal ping check interval  
#define IPING_FREQUENCY 1  // Ping frequency multiplier
#define GATEWAY_SECONDS 1  // Gateway check interval
#define SOFIA_QUEUE_SIZE 50000  // Event queue size
#define SOFIA_NAT_SESSION_TIMEOUT 90  // 90 seconds NAT session timeout
#define SOFIA_MAX_ACL 100  // Max ACL entries per profile
#define ROUTE_MAX_HEADERS 20  // Max route headers processed
#define ROUTE_ENCODED_HEADER_MAX_CHARS (1024 * 3)  // 3KB max encoded route header
#define MAX_CODEC_CHECK_FRAMES 50  // Max frames analyzed for codec detection
#define MAX_MISMATCH_FRAMES 5  // Max codec mismatch frames allowed
#define SOFIA_CUSTOM_NUA_EVENT_REFER 9999  // Custom NUA event for REFER
#define SOFIA_MAX_MSG_QUEUE 64  // Max message queue workers
#define SOFIA_MSG_QUEUE_SIZE 1000  // Size per message queue
#define SOFIA_MAX_REG_ALGS 7  // RFC8760 max registration algorithms
#define MAX_RTPIP 50  // Max RTP IP addresses per profile
```

### RFC 7989 Session ID
```c
#define RFC7989_SESSION_UUID_LEN 32  // Session UUID length
#define RFC7989_SESSION_UUID_NULL "00000000000000000000000000000000"  // NULL UUID
```

### Transport Types
```c
typedef enum {
    SOFIA_TRANSPORT_UNKNOWN = 0,
    SOFIA_TRANSPORT_UDP,
    SOFIA_TRANSPORT_TCP,
    SOFIA_TRANSPORT_TCP_TLS,
    SOFIA_TRANSPORT_SCTP,
    SOFIA_TRANSPORT_WS,  // WebSocket
    SOFIA_TRANSPORT_WSS  // WebSocket Secure
} sofia_transport_t;
```

## SIP Protocol Implementation

### Authentication Response Handling
```c
// Line 2442 in sofia.c
if ((status == 401 || status == 407 || status == 403) && sofia_private) {
  // Special auth failure handling
}
```

### 403 Forbidden Override
```c
// PFLAG_BLIND_AUTH_REPLY_403: Send 403 instead of challenging
if (sofia_test_pflag(profile, PFLAG_BLIND_AUTH_REPLY_403)) {
    nua_respond(nh, SIP_403_FORBIDDEN, TAG_END());
}
```

### Method Dispatching Critical Points
```c
case nua_i_invite:
    if (sofia_private->is_call > 1) {
    sofia_handle_sip_i_reinvite(...);  // Re-INVITE handling
    } else {
    sofia_private->is_call++;
    }
    break;

case nua_i_refer:
    if (switch_true(profile->proxy_refer_events)) {
    sofia_proxy_sip_i_refer(...);
    } else {
    sofia_handle_sip_i_refer(...);
    }
    break;
```

### Special Header Detection
```c
#define sofia_test_extra_headers(val) \
    (((!strncasecmp(val, "X-", 2) && strncasecmp(val, "X-FS-", 5)) || \
      !strncasecmp(val, "P-", 2) || !strncasecmp(val, "On", 2)) ? 1 : 0)
```
Detects:
- X- headers (except X-FS- FreeSWITCH specific)
- P- headers (proprietary/private)
- Headers starting with "On" (OnAnswer, etc.)

## State Machine Implementation

### Profile Flags (PFLAGS) - Critical Behavior Control

#### Authentication Flags
```c
PFLAG_AUTH_CALLS,  // Authenticate all calls
PFLAG_AUTH_MESSAGES,  // Authenticate MESSAGE requests
PFLAG_AUTH_ALL,  // Authenticate everything
PFLAG_BLIND_REG,  // Allow blind registration without auth
PFLAG_BLIND_AUTH,  // Allow blind authentication
PFLAG_BLIND_AUTH_ENFORCE_RESULT, // Enforce blind auth results
PFLAG_BLIND_AUTH_REPLY_403,  // Reply 403 instead of challenge
PFLAG_AUTH_SUBSCRIPTIONS,  // Authenticate SUBSCRIBE requests
PFLAG_AUTH_REQUIRE_USER,  // Require user in auth
PFLAG_AUTH_CALLS_ACL_ONLY,  // Auth calls using ACL only
```

#### NAT/Network Flags
```c
PFLAG_AGGRESSIVE_NAT_DETECTION,  // Enhanced NAT detection
PFLAG_RECIEVED_IN_NAT_REG_CONTACT, // Use received in NAT reg contact
PFLAG_NAT_OPTIONS_PING,  // NAT keep-alive via OPTIONS
PFLAG_UDP_NAT_OPTIONS_PING,  // UDP-specific NAT pinging
PFLAG_AUTO_NAT,  // Automatic NAT detection
PFLAG_TLS_ALWAYS_NAT,  // Treat TLS as always NATed
PFLAG_TCP_ALWAYS_NAT,  // Treat TCP as always NATed
```

#### 3PCC (Third Party Call Control) Flags
```c
PFLAG_3PCC,  // Enable 3PCC
PFLAG_3PCC_PROXY,  // 3PCC proxy mode
PFLAG_3PCC_REINVITE_BRIDGED_ON_ACK, // Re-INVITE on ACK in 3PCC
```

### Tech Flags (TFLAGS) - Session State
```c
TFLAG_CHANGE_MEDIA,  // Media change in progress
TFLAG_NOSDP_REINVITE,  // Re-INVITE without SDP
TFLAG_SIP_HOLD_INACTIVE,  // SIP hold inactive state
TFLAG_3PCC_HAS_ACK,  // 3PCC ACK received
TFLAG_ENABLE_SOA,  // SDP Offer/Answer enabled
TFLAG_RECOVERED,  // Call recovered from crash
TFLAG_3PCC_INVITE,  // 3PCC INVITE sent
TFLAG_PASS_ACK,  // Pass ACK to application
TFLAG_KEEPALIVE,  // Keepalive active
```

## Authentication and Security

### Digest Algorithms (RFC 8760)
```c
typedef enum {
    ALG_MD5 = (1 << 0),  // MD5 (deprecated)
    ALG_SHA256 = (1 << 1),  // SHA-256
    ALG_SHA512 = (1 << 2),  // SHA-512
    ALG_NONE = (1 << 3),  // No algorithm specified
} sofia_auth_algs_t;

#define SOFIA_MAX_REG_ALGS 7  // Max algorithms per registration
```

### Authentication State
```c
typedef enum {
    AUTH_OK,  // Authentication successful
    AUTH_FORBIDDEN,  // Authentication forbidden
    AUTH_STALE,  // Stale nonce
    AUTH_RENEWED,  // Nonce renewed
} auth_res_t;
```

### Registration State Machine
```c
typedef enum {
    REG_STATE_UNREGED,  // Not registered
    REG_STATE_TRYING,  // Registration attempt in progress
    REG_STATE_REGISTER,  // Registering
    REG_STATE_REGED,  // Successfully registered
    REG_STATE_UNREGISTER,  // Unregistering
    REG_STATE_FAILED,  // Registration failed
    REG_STATE_FAIL_WAIT,  // Waiting after failure
    REG_STATE_EXPIRED,  // Registration expired
    REG_STATE_NOREG,  // No registration required
    REG_STATE_DOWN,  // Gateway down
    REG_STATE_TIMEOUT,  // Registration timeout
    REG_STATE_LAST  // Sentinel
} reg_state_t;
```

## NAT and Transport Handling

### NAT Detection Logic
```c
// sofia_glue.c
int sofia_glue_check_nat(sofia_profile_t *profile, const char *network_ip)
{
    return (profile->extsipip &&
    !switch_check_network_list_ip(network_ip, "loopback.auto") &&
    !switch_check_network_list_ip(network_ip, profile->local_network));
}
```
NAT detected if:
1. External SIP IP configured
2. Source IP not loopback
3. Source IP not in local network range

### Keep-alive Types
```c
typedef enum {
    KA_MESSAGE,  // Keep-alive via MESSAGE
    KA_INFO  // Keep-alive via INFO
} ka_type_t;
```

## Special Cases and Workarounds

### NDLB (No Doubt Left Behind) Compatibility
```c
typedef enum {
    PFLAG_NDLB_TO_IN_200_CONTACT = (1 << 0),  // Put To in 200 Contact
    PFLAG_NDLB_BROKEN_AUTH_HASH = (1 << 1),  // Broken auth hash handling
    PFLAG_NDLB_EXPIRES_IN_REGISTER_RESPONSE = (1 << 2)  // Expires in reg response
} sofia_NDLB_t;
```

### Presence Types
```c
typedef enum {
    PRES_TYPE_NONE = 0,  // No presence
    PRES_TYPE_FULL = 1,  // Full presence support
    PRES_TYPE_PASSIVE = 2,  // Passive presence
    PRES_TYPE_PNP = 3  // Plug-and-Play presence
} sofia_presence_type_t;
```

### Media Options
```c
typedef enum {
    MEDIA_OPT_NONE = 0,
    MEDIA_OPT_MEDIA_ON_HOLD = (1 << 0),  // Media handling on hold
    MEDIA_OPT_BYPASS_AFTER_ATT_XFER = (1 << 1),  // Bypass after attended transfer
    MEDIA_OPT_BYPASS_AFTER_HOLD = (1 << 2)  // Bypass after hold
} sofia_media_options_t;
```

## Response Code Patterns

### 200 OK Variations
```c
// Basic 200 OK for NOTIFY
nua_respond(nh, SIP_200_OK, NUTAG_WITH_THIS_MSG(de->data->e_msg), TAG_END());

// 200 OK with Session-ID (RFC 7989)
nua_respond(nh, SIP_200_OK, TAG_IF(!zstr(session_id_header), SIPTAG_HEADER_STR(session_id_header)), TAG_END());

// 200 OK for re-INVITE with custom headers
nua_respond(tech_pvt->nh, SIP_200_OK, SIPTAG_CONTACT_STR(contact_str), TAG_IF(!zstr(session_id_header), SIPTAG_HEADER_STR(session_id_header)), TAG_END());
```

### Error Response Patterns
```c
// 488 Not Acceptable Here - codec/media negotiation failures
nua_respond(nh, SIP_488_NOT_ACCEPTABLE, NUTAG_WITH_THIS_MSG(de->data->e_msg), TAG_END());

// 491 Request Pending - prevent SIP glare conditions
nua_respond(tech_pvt->nh, SIP_491_REQUEST_PENDING, TAG_END());

// 422 Session Timer Too Small - session timer negotiation
nua_respond(nh, SIP_422_SESSION_TIMER_TOO_SMALL, SIPTAG_HEADER_STR(buf), TAG_END());

// 503 Service Unavailable with Retry-After - overload protection
nua_respond(nh, 503, "Maximum Calls In Progress", 
    SIPTAG_RETRY_AFTER_STR(profile->unavailable_retry_after_timeout), 
    NUTAG_WITH_THIS(nua), TAG_END());
```

### Wait for Reply Mechanism
```c
void sofia_wait_for_reply(struct private_object *tech_pvt, nua_event_t event, uint32_t timeout)
{
    time_t exp = switch_epoch_time_now(NULL) + timeout;
    tech_pvt->want_event = event;
    
    while (switch_channel_ready(tech_pvt->channel) && tech_pvt->want_event && 
    switch_epoch_time_now(NULL) < exp) {
    switch_yield(100000);  // Yield for 100ms
    }
    
    tech_pvt->want_event = 0;
}
```

## Multi-Algorithm Authentication (RFC 8760)
```c
// Authentication challenge with multiple algorithms
if (profile->rfc8760_algs_count) {
    nua_respond(nh, SIP_407_PROXY_AUTH_REQUIRED,
    TAG_IF(msg, NUTAG_WITH_THIS_MSG(msg)),
    TAG_IF(auth_str_rfc8760[0], SIPTAG_PROXY_AUTHENTICATE_STR(auth_str_rfc8760[0])), 
    TAG_IF(auth_str_rfc8760[1], SIPTAG_PROXY_AUTHENTICATE_STR(auth_str_rfc8760[1])), 
    TAG_IF(auth_str_rfc8760[2], SIPTAG_PROXY_AUTHENTICATE_STR(auth_str_rfc8760[2])), 
  // ... up to 7 algorithms
    TAG_END());
}
```

## Codec Validation
```c
#define MAX_CODEC_CHECK_FRAMES 50  // Max frames analyzed for codec detection
#define MAX_MISMATCH_FRAMES 5  // Max codec mismatch frames allowed
```
- System analyzes up to 50 frames for codec compatibility
- If >5 frames don't match negotiated codec, call may terminate

## Performance Controls

### Call Volume Management
```c
// Max call limiting with retry-after
if (sess_count >= sess_max || !sofia_test_pflag(profile, PFLAG_RUNNING) || 
    !switch_core_ready_inbound()) {
    nua_respond(nh, 503, "Maximum Calls In Progress", 
    SIPTAG_RETRY_AFTER_STR(profile->unavailable_retry_after_timeout), 
    NUTAG_WITH_THIS(nua), TAG_END());
}

// Message queue overload protection
if (switch_queue_size(mod_sofia_globals.msg_queue) > (unsigned int)critical) {
    nua_respond(nh, 503, "System Busy", 
    SIPTAG_RETRY_AFTER_STR(profile->unavailable_retry_after_timeout), 
    NUTAG_WITH_THIS(nua), TAG_END());
}
```

## NAT Structure and Contact Generation

### NAT Parse Structure
```c
typedef struct {
    char network_ip[80];  // Detected network IP
    int network_port;  // Network port
    const char *is_nat;  // NAT detection result string
    int is_auto_nat;  // Auto NAT detection flag
    int fs_path;  // FreeSWITCH path header flag
} sofia_nat_parse_t;
```

### Critical Subscription Cleanup
```c
// 481 Call/Transaction Does Not Exist - cleanup orphaned subscriptions
if (status == 481 && sip && !sip->sip_retry_after && sip->sip_call_id && 
    (!sofia_private || !sofia_private->is_call)) {
    char *sql = switch_mprintf("delete from sip_subscriptions where call_id='%q'", 
    sip->sip_call_id->i_id);
  // Execute cleanup SQL
}
```

## Advanced Features

### RFC 7989 Session ID
```c
PFLAG_RFC7989_SESSION_ID,  // Enable RFC 7989
PFLAG_RFC7989_FORCE_OLD,  // Force old session ID format
```

### STIR/SHAKEN Variables
```c
const char *stir_shaken_as_key;  // Authentication service key
const char *stir_shaken_as_url;  // Authentication service URL
const char *stir_shaken_vs_ca_dir;  // Verification service CA directory
```

## Timing Constants

### Session Timers
```c
uint32_t session_timeout;  // Session timeout in seconds
uint32_t minimum_session_expires;  // Min session expires value
enum nua_session_refresher session_refresher;  // Who refreshes session
```

### SIP Timers (RFC 3261)
```c
uint32_t timer_t1;  // T1 timer (RTT estimate)
uint32_t timer_t1x64;  // T1 * 64 (max retransmit interval)
uint32_t timer_t2;  // T2 timer (non-INVITE max retransmit)
uint32_t timer_t4;  // T4 timer (max msg duration)
```

### TLS Timeouts
```c
int tls_connect_timeout_ms;  // TLS connection timeout in ms
unsigned int tls_timeout;  // General TLS timeout
uint32_t tls_orq_connect_timeout;  // TLS ORQ connection timeout
```

## Critical Debugging Points

1. **Authentication Flow**: Check `auth_res_t` return values and nonce handling
2. **NAT Detection**: Examine `sofia_nat_parse_t` contents
3. **Transport Issues**: Verify `sofia_transport_t` conversions
4. **Call State**: Monitor `TFLAG_*` and `PFLAG_*` combinations
5. **Response Codes**: Watch for 488, 491, 422, 503 patterns
6. **Queue Overflow**: Monitor message queue sizes vs limits
7. **Codec Validation**: Track frame analysis and mismatch counts