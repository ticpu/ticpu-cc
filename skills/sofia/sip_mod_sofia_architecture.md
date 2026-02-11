# FreeSWITCH mod_sofia.c Technical Documentation

**Location:** `/mnt/bcachefs/home/jerome/GIT/freeswitch/freeswitch/src/mod/endpoints/mod_sofia/mod_sofia.c`
**Lines:** 7,159

## Core Architecture

```c
SWITCH_MODULE_DEFINITION(mod_sofia, mod_sofia_load, mod_sofia_shutdown, NULL);
```

## Key Data Structures

### `mod_sofia_globals`
```c
struct mod_sofia_globals {
    switch_memory_pool_t *pool;
    switch_hash_t *profile_hash;
    switch_hash_t *gateway_hash;
    switch_mutex_t *hash_mutex;
    uint32_t callid;  // Call-ID counter
    int32_t running;
    int32_t threads;
    int cpu_count;
    int max_msg_queues;
    switch_queue_t *presence_queue;
    switch_queue_t *msg_queue;
    switch_thread_t *msg_queue_thread[SOFIA_MAX_MSG_QUEUE];
};
```

### `sofia_profile`
```c
struct sofia_profile {
    char *name;
    char *domain_name;
    char *dialplan;
    char *context;
    char *sipip;
    char *url;
    char *bindurl;
    char *user_agent;
    switch_port_t sip_port;
    switch_port_t tls_sip_port;
    nua_t *nua;  // Sofia NUA stack instance
    su_root_t *s_root;  // Sofia event loop root
    switch_memory_pool_t *pool;
    sofia_gateway_t *gateways;
    uint8_t pflags[PFLAG_MAX];
    uint8_t flags[TFLAG_MAX];
};
```

### `private_object`
```c
struct private_object {
    sofia_private_t *sofia_private;
    uint8_t flags[TFLAG_MAX];
    switch_core_session_t *session;
    switch_channel_t *channel;
    switch_media_handle_t *media_handle;
    sofia_profile_t *profile;
    char *reply_contact;
    char *from_uri;
    char *to_uri;
    char *callid;
    char *contact_url;
    nua_handle_t *nh;
    nua_handle_t *nh2;  // Secondary NUA handle for transfers
    sofia_transport_t transport;
    uint32_t session_timeout;
};
```

### `sofia_gateway`
```c
struct sofia_gateway {
    char *name;
    char *username;
    char *password;
    char *realm;
    char *from_user;
    char *from_domain;
    char *proxy;
    char *register_proxy;
    reg_state_t state;
    time_t retry;  // Next retry time
    uint32_t freq;  // Registration frequency in seconds
};
```

## Configuration Constants

```c
#define SOFIA_QUEUE_SIZE 50000  // Event queue size
#define SOFIA_MAX_ACL 100  // Maximum ACL entries
#define SOFIA_NAT_SESSION_TIMEOUT 90  // 90 seconds for NAT sessions
#define ROUTE_MAX_HEADERS 20  // Max headers processed in Route
#define SOFIA_DEFAULT_PORT "5060"
#define SOFIA_DEFAULT_TLS_PORT "5061"
#define SOFIA_MAX_MSG_QUEUE 64  // Max message queue workers
#define SOFIA_MSG_QUEUE_SIZE 1000  // Messages per queue
#define SOFIA_MAX_REG_ALGS 7  // Max auth algorithms per RFC8760
```

## Channel Variables

### SIP Header Prefixes
```c
#define SOFIA_SIP_HEADER_PREFIX "sip_h_"  // General headers
#define SOFIA_SIP_INFO_HEADER_PREFIX "sip_info_h_"  // INFO headers
#define SOFIA_SIP_RESPONSE_HEADER_PREFIX "sip_rh_"  // Response headers
#define SOFIA_SIP_BYE_HEADER_PREFIX "sip_bye_h_"  // BYE headers
#define SOFIA_SIP_PROGRESS_HEADER_PREFIX "sip_ph_"  // Progress headers
#define SOFIA_MULTIPART_PREFIX "sip_mp_"  // Multipart content
```

### Key Variables
- `sip_watch_headers` - Headers monitored for changes
- `sip_cid_in_1xx` - Include caller ID in 1xx responses
- `sip_gateway_name` - Gateway used for outbound calls
- `sip_ignore_remote_cause` - Ignore remote hangup cause
- `disable_q850_reason` - Disable Q.850 reason codes
- `ignore_completed_elsewhere` - Ignore 200 OK responses after timeout
- `sofia_session_timeout` - Session timer value in seconds

## Flags

### Profile Flags (PFLAGS)
```c
PFLAG_AUTH_CALLS,  // Authenticate calls
PFLAG_AUTH_MESSAGES,  // Authenticate MESSAGE requests
PFLAG_AUTH_ALL,  // Authenticate all requests
PFLAG_MULTIREG,  // Multiple registrations per AOR
PFLAG_TLS,  // TLS enabled
PFLAG_AGGRESSIVE_NAT_DETECTION,  // Enhanced NAT detection
PFLAG_3PCC,  // 3rd Party Call Control
PFLAG_DISABLE_100REL,  // Disable reliable provisional responses
PFLAG_NAT_OPTIONS_PING,  // NAT keepalive via OPTIONS
PFLAG_T38_PASSTHRU,  // T.38 fax passthrough
PFLAG_AUTO_NAT,  // Automatic NAT handling
PFLAG_LOG_AUTH_FAIL,  // Log authentication failures
PFLAG_TRACK_CALLS,  // Track call statistics
PFLAG_PARSE_ALL_INVITE_HEADERS,  // Parse all INVITE headers into variables
```

### Technical Flags (TFLAGS)
```c
TFLAG_IO,  // I/O ready
TFLAG_OUTBOUND,  // Outbound call
TFLAG_HUP,  // Hangup initiated
TFLAG_RTP,  // RTP active
TFLAG_BYE,  // BYE sent/received
TFLAG_ANS,  // Call answered
TFLAG_EARLY_MEDIA,  // Early media active
TFLAG_3PCC,  // 3PCC mode active
TFLAG_REFER,  // REFER in progress
TFLAG_NAT,  // NAT detected
TFLAG_SIP_HOLD,  // SIP hold state
TFLAG_LATE_NEGOTIATION,  // Late media negotiation
TFLAG_PROXY_MEDIA,  // Proxy media mode
TFLAG_REINVITED,  // Re-INVITE processed
```

## Core Functions

### State Machine Handlers
```c
static switch_status_t sofia_on_init(switch_core_session_t *session);
static switch_status_t sofia_on_routing(switch_core_session_t *session);
static switch_status_t sofia_on_execute(switch_core_session_t *session);
static switch_status_t sofia_on_exchange_media(switch_core_session_t *session);
static switch_status_t sofia_acknowledge_call(switch_core_session_t *session);
```

### Media Handlers
```c
static switch_status_t sofia_read_frame(switch_core_session_t *session, 
    switch_frame_t **frame, 
    switch_io_flag_t flags, int stream_id);
static switch_status_t sofia_write_frame(switch_core_session_t *session, 
    switch_frame_t *frame, 
    switch_io_flag_t flags, int stream_id);
```

## Sofia-SIP Integration

### Core Components
```c
#include <sofia-sip/nua.h>  // High-level SIP user agent API
#include <sofia-sip/sip_status.h>
#include <sofia-sip/sip_protos.h>
#include <sofia-sip/sip_parser.h>
#include <sofia-sip/sdp.h>  // SDP handling
#include <sofia-sip/auth_module.h>   // HTTP Digest authentication
#include <sofia-sip/su_md5.h>
#include <sofia-sip/tport_tag.h>  // Transport layer (UDP/TCP/TLS/WS)
```

### Event Dispatch
```c
typedef struct sofia_dispatch_event_s {
    nua_saved_event_t event[1];
    nua_handle_t *nh;
    nua_event_data_t const *data;
    sip_t *sip;
    nua_t *nua;
    sofia_profile_t *profile;
    switch_core_session_t *session;
} sofia_dispatch_event_t;
```

## Events

### Sofia-Specific Events
```c
#define MY_EVENT_REGISTER "sofia::register"
#define MY_EVENT_PRE_REGISTER "sofia::pre_register"
#define MY_EVENT_REGISTER_ATTEMPT "sofia::register_attempt"
#define MY_EVENT_REGISTER_FAILURE "sofia::register_failure"
#define MY_EVENT_UNREGISTER "sofia::unregister"
#define MY_EVENT_EXPIRE "sofia::expire"
#define MY_EVENT_GATEWAY_STATE "sofia::gateway_state"
#define MY_EVENT_NOTIFY_REFER "sofia::notify_refer"
#define MY_EVENT_REINVITE "sofia::reinvite"
#define MY_EVENT_RECOVERY "sofia::recovery_recv"
#define MY_EVENT_ERROR "sofia::error"
#define MY_EVENT_PROFILE_START "sofia::profile_start"
#define MY_EVENT_BYE_RESPONSE "sofia::bye_response"
```

## Debugging

### Debug Variables
```c
int debug_presence;  // Presence debugging level
int debug_sla;  // SLA debugging level
int tracelevel;  // Sofia trace level (0-9)
char *capture_server;  // HEP capture server URL
```

### Key Techniques
1. **SIP Tracing**: Enable with `tracelevel` or `sip_trace` profile parameter
2. **HEP Capture**: Configure `capture_server` for Homer integration
3. **Channel Variables**: Use `uuid_dump <uuid>` to inspect all variables
4. **Profile Status**: `sofia status profile <name>` for detailed info
5. **Gateway Monitoring**: `sofia status gateway <name>`

### CLI Commands
```bash
sofia profile <name> start|stop|restart|rescan
sofia profile <name> siptrace on|off
sofia status profile <name> reg
sofia loglevel all 9
sofia tracelevel debug
```

### Common Issues
- **Registration**: Check `MY_EVENT_REGISTER_FAILURE` events and ACL settings
- **NAT**: Review `PFLAG_AGGRESSIVE_NAT_DETECTION` and `ext-*-ip` config
- **Media**: Examine SDP offer/answer and `TFLAG_LATE_NEGOTIATION`
- **Transport**: Verify TCP/TLS config and certificate validity

## Files
- `/freeswitch/src/mod/endpoints/mod_sofia/mod_sofia.c` - Main implementation
- `/freeswitch/src/mod/endpoints/mod_sofia/sofia_glue.c` - Integration glue
- `/freeswitch/src/mod/endpoints/mod_sofia/sofia_presence.c` - Presence
- `/freeswitch/src/mod/endpoints/mod_sofia/sofia_reg.c` - Registration
- `/conf/sip_profiles/internal.xml` - Default internal profile
- `/conf/sip_profiles/external.xml` - Default external profile