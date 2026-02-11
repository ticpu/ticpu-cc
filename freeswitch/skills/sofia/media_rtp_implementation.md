# FreeSWITCH RTP Implementation Documentation

## Overview and Purpose

The `switch_rtp.c` file implements FreeSWITCH's comprehensive Real-Time Transport Protocol (RTP) functionality. This 9,100+ line implementation provides:

- **RTP packet handling**: Complete transmission and reception of RTP media streams
- **SRTP encryption/decryption**: Secure RTP with encryption and authentication
- **DTMF handling**: RFC2833 DTMF tone detection and generation
- **Jitter buffer management**: Adaptive buffering for smooth media playback
- **RTP statistics**: Comprehensive quality metrics and performance monitoring
- **DTLS integration**: Secure key exchange for WebRTC compatibility
- **ICE support**: Interactive Connectivity Establishment for NAT traversal
- **RTCP functionality**: Real-time Control Protocol for quality feedback

This implementation is designed to handle real-world VoIP scenarios with extensive error handling, debugging capabilities, and compatibility workarounds for various network conditions and equipment.

## Key Data Structures and Types

### Main RTP Session Structure (`struct switch_rtp`)

The core RTP session structure contains all state needed for an RTP stream:

```c
struct switch_rtp {
  // Network sockets for RTP and RTCP
    switch_socket_t *sock_input, *sock_output, *rtcp_sock_input, *rtcp_sock_output;
    switch_pollfd_t *read_pollfd, *rtcp_read_pollfd;
    switch_pollfd_t *jb_pollfd;

  // Address information
    switch_sockaddr_t *local_addr, *rtcp_local_addr;
    switch_sockaddr_t *remote_addr, *rtcp_remote_addr;

  // Message buffers
    rtp_msg_t send_msg;
    rtcp_msg_t rtcp_send_msg;
    rtp_msg_t recv_msg;
    rtcp_msg_t rtcp_recv_msg;

  // SRTP contexts for encryption
    srtp_ctx_t *send_ctx[2];
    srtp_ctx_t *recv_ctx[2];
    srtp_policy_t send_policy[2];
    srtp_policy_t recv_policy[2];

  // DTLS for secure key exchange
    switch_dtls_t *dtls;
    switch_dtls_t *rtcp_dtls;

  // RTP sequence and timestamp tracking
    uint16_t seq;
    uint32_t ssrc;
    uint32_t remote_ssrc;
    uint32_t ts;
    uint32_t last_write_ts;
    uint32_t last_read_ts;

  // Jitter buffers
    switch_jb_t *jb;  // Audio jitter buffer
    switch_jb_t *vb;  // Video jitter buffer
    switch_jb_t *vbw;   // Video jitter buffer (write)

  // DTMF/RFC2833 data
    struct switch_rtp_rfc2833_data dtmf_data;

  // VAD (Voice Activity Detection) data
    struct switch_rtp_vad_data vad_data;

  // ICE for NAT traversal
    switch_rtp_ice_t ice;
    switch_rtp_ice_t rtcp_ice;

  // Statistics and quality metrics
    switch_rtp_stats_t stats;
    switch_rtcp_video_stats_t rtcp_vstats;

  // Flags controlling behavior
    uint32_t flags[SWITCH_RTP_FLAG_INVALID];

  // Mutexes for thread safety
    switch_mutex_t *flag_mutex;
    switch_mutex_t *read_mutex;
    switch_mutex_t *write_mutex;
    switch_mutex_t *ice_mutex;
};
```

### RTP Message Structure (`rtp_msg_t`)

```c
typedef struct {
    srtp_hdr_t header;  // Standard RTP header
    char body[SWITCH_RTP_MAX_BUF_LEN+4+sizeof(char *)]; // Payload data
    switch_rtp_hdr_ext_t *ext;  // RTP header extensions
    char *ebody;  // Extended body pointer
} rtp_msg_t;
```

### RFC2833 DTMF Structure (`struct switch_rtp_rfc2833_data`)

```c
struct switch_rtp_rfc2833_data {
    switch_queue_t *dtmf_queue;  // Queue for incoming DTMF
    char out_digit;  // Current outgoing digit
    unsigned char out_digit_packet[4];  // DTMF packet being sent
    unsigned int out_digit_sofar;  // Progress of current digit
    unsigned int out_digit_dur;  // Duration of current digit
    uint16_t in_digit_seq;  // Sequence of incoming DTMF
    uint32_t in_digit_ts;  // Timestamp of incoming DTMF
    uint32_t in_digit_sanity;  // Sanity counter for DTMF
    switch_queue_t *dtmf_inqueue;  // Input DTMF queue
    switch_mutex_t *dtmf_mutex;  // DTMF thread protection
    uint8_t in_digit_queued;  // Flag for queued digits
};
```

### ICE Structure (`switch_rtp_ice_t`)

```c
typedef struct {
    char *ice_user;  // ICE username
    char *user_ice;  // Local ICE username
    char *pass;  // ICE password
    switch_sockaddr_t *addr;  // ICE candidate address
    ice_t *ice_params;  // ICE parameters
    ice_proto_t proto;  // Protocol (UDP/TCP)
    uint8_t sending;  // Currently sending ICE
    uint8_t ready;  // ICE ready state
    uint8_t initializing;  // ICE initialization state
    int missed_count;  // Missed ICE packets
    switch_time_t last_ok;  // Last successful ICE
} switch_rtp_ice_t;
```

### DTLS Structure (`switch_dtls_t`)

```c
typedef struct switch_dtls_s {
    SSL_CTX *ssl_ctx;  // OpenSSL context
    SSL *ssl;  // SSL connection
    BIO *read_bio;  // Read BIO
    BIO *write_bio;  // Write BIO
    dtls_fingerprint_t *local_fp;  // Local certificate fingerprint
    dtls_fingerprint_t *remote_fp;  // Remote certificate fingerprint
    dtls_state_t state;  // Current DTLS state
    dtls_type_t type;  // DTLS type (RTP/RTCP)
    switch_socket_t *sock_output;  // Output socket
    switch_sockaddr_t *remote_addr;  // Remote address
    struct switch_rtp *rtp_session;  // Parent RTP session
    int mtu;  // Maximum transmission unit
} switch_dtls_t;
```

## Core RTP Functions

### Session Management

#### `switch_rtp_create()` - Create RTP Session
Creates a new RTP session with specified parameters:

```c
SWITCH_DECLARE(switch_status_t) switch_rtp_create(switch_rtp_t **new_rtp_session,
    switch_payload_t payload,
    uint32_t samples_per_interval,
    uint32_t ms_per_packet,
    switch_rtp_flag_t flags[SWITCH_RTP_FLAG_INVALID],
    char *timer_name,
    const char **err,
    switch_memory_pool_t *pool,
    switch_port_t bundle_internal_port,
    switch_port_t bundle_external_port);
```

**Key functionality:**
- Allocates and initializes RTP session structure
- Sets up mutexes for thread safety
- Initializes sequence numbers and SSRC
- Configures payload type and timing parameters
- Sets up debugging and statistics tracking

#### `switch_rtp_destroy()` - Destroy RTP Session
Cleanly destroys an RTP session:

```c
SWITCH_DECLARE(void) switch_rtp_destroy(switch_rtp_t **rtp_session);
```

**Key functionality:**
- Closes all network sockets
- Destroys DTLS contexts
- Cleans up SRTP contexts
- Frees jitter buffers
- Destroys mutexes and memory pools

### Address and Network Configuration

#### `switch_rtp_set_local_address()` - Set Local Address
Configures the local network address for RTP:

```c
SWITCH_DECLARE(switch_status_t) switch_rtp_set_local_address(switch_rtp_t *rtp_session,
    const char *host,
    switch_port_t port,
    const char **err);
```

#### `switch_rtp_set_remote_address()` - Set Remote Address
Configures the remote network address for RTP:

```c
SWITCH_DECLARE(switch_status_t) switch_rtp_set_remote_address(switch_rtp_t *rtp_session,
    const char *host,
    switch_port_t port,
    switch_port_t remote_rtcp_port,
    switch_bool_t change_adv_addr,
    const char **err);
```

**Key functionality:**
- Resolves hostname to IP address
- Creates socket addresses for RTP and RTCP
- Handles IPv4/IPv6 protocol families
- Updates ICE candidate information if needed

### Packet Reading and Writing

#### `switch_rtp_read()` - Read RTP Packet
Main function for reading RTP packets:

```c
SWITCH_DECLARE(switch_status_t) switch_rtp_read(switch_rtp_t *rtp_session,
    void *data,
    uint32_t *datalen,
    switch_payload_t *payload_type,
    switch_frame_flag_t *flags,
    switch_io_flag_t io_flags);
```

**Key functionality:**
- Receives packets from network socket
- Handles SRTP decryption
- Processes RFC2833 DTMF packets
- Manages jitter buffer
- Updates RTP statistics
- Handles out-of-order packet detection
- Processes RTCP packets

#### `switch_rtp_write_frame()` - Write RTP Frame
Writes an RTP frame to the network:

```c
SWITCH_DECLARE(int) switch_rtp_write_frame(switch_rtp_t *rtp_session,
    switch_frame_t *frame);
```

**Key functionality:**
- Constructs RTP header with sequence and timestamp
- Handles SRTP encryption
- Manages payload type changes
- Implements adaptive jitter buffer timing
- Handles RTCP transmission timing
- Updates transmission statistics

## SRTP Functionality

### SRTP Context Management

The implementation supports dual SRTP contexts for seamless key rollover:

```c
// SRTP contexts - index 0 and 1 for key rollover
srtp_ctx_t *send_ctx[2];
srtp_ctx_t *recv_ctx[2];
srtp_policy_t send_policy[2];
srtp_policy_t recv_policy[2];
```

#### `switch_rtp_add_crypto_key()` - Add SRTP Key
Adds an SRTP encryption key:

```c
SWITCH_DECLARE(switch_status_t) switch_rtp_add_crypto_key(switch_rtp_t *rtp_session,
    switch_rtp_crypto_direction_t direction,
    uint32_t index,
    switch_secure_settings_t *ssec);
```

**Key functionality:**
- Parses crypto suite (AES_CM_128_HMAC_SHA1_80, etc.)
- Derives master key and salt from key material
- Creates and configures SRTP policy
- Initializes SRTP context
- Supports key rollover with dual contexts

### SRTP Encryption/Decryption

The implementation automatically handles SRTP processing:

**Encryption (Outbound):**
- Applied in `rtp_common_write()` function
- Encrypts RTP payload and adds authentication tag
- Handles sequence number rollover protection

**Decryption (Inbound):**
- Applied in packet reception
- Verifies authentication tag
- Decrypts payload data
- Tracks and reports SRTP errors

### Error Handling

SRTP errors are tracked and logged:

```c
uint32_t srtp_errs[2];  // RTP SRTP errors per context
uint32_t srctp_errs[2];   // RTCP SRTP errors per context

#define WARN_SRTP_ERRS 10   // Warning threshold
#define MAX_SRTP_ERRS 100   // Maximum errors before reset
```

## DTMF and RFC2833 Handling

### RFC2833 Packet Structure

DTMF tones are transmitted as RFC2833 packets with this structure:
- **Event (8 bits)**: DTMF digit code (0-15)
- **E bit**: End of event flag
- **R bit**: Reserved (must be 0)
- **Volume (6 bits)**: Signal volume
- **Duration (16 bits)**: Event duration in timestamp units

### DTMF Detection (`handle_rfc2833()`)

The RFC2833 handler processes incoming DTMF packets:

```c
static handle_rfc2833_result_t handle_rfc2833(switch_rtp_t *rtp_session,
    switch_size_t bytes,
    int *do_cng);
```

**Key functionality:**
- Validates RFC2833 packet format
- Handles payload offset corrections for broken implementations
- Processes start, continuation, and end events
- Maintains sanity counters to prevent infinite DTMF
- Queues DTMF events for application processing
- Handles sequence number validation and duplicate detection

### DTMF Generation (`do_2833()`)

Generates outgoing RFC2833 DTMF packets:

```c
static void do_2833(switch_rtp_t *rtp_session);
```

**Key functionality:**
- Creates RFC2833 packet format
- Manages digit duration and repetition
- Handles proper end-of-digit signaling
- Maintains timestamp consistency
- Implements inter-digit delay timing

### DTMF Queue Management

DTMF events are queued for asynchronous processing:

```c
SWITCH_DECLARE(switch_status_t) switch_rtp_queue_rfc2833(switch_rtp_t *rtp_session,
    const switch_dtmf_t *dtmf);
SWITCH_DECLARE(switch_size_t) switch_rtp_dequeue_dtmf(switch_rtp_t *rtp_session,
    switch_dtmf_t *dtmf);
```

## Jitter Buffer Implementation

### Adaptive Jitter Buffer

The implementation includes sophisticated adaptive jitter buffering:

#### Audio Jitter Buffer (`switch_jb_t *jb`)
- Handles audio packet timing variations
- Dynamically adjusts buffer size based on network conditions
- Implements packet loss concealment
- Maintains timestamp continuity

#### Video Jitter Buffer (`switch_jb_t *vb`, `switch_jb_t *vbw`)
- Separate read and write video buffers
- Frame-based buffering for video streams
- Handles keyframe detection and ordering
- Supports configurable buffer depths

### Jitter Buffer Functions

#### `switch_rtp_activate_jitter_buffer()` - Enable Jitter Buffer
```c
SWITCH_DECLARE(switch_status_t) switch_rtp_activate_jitter_buffer(switch_rtp_t *rtp_session,
    uint32_t queue_frames,
    uint32_t max_queue_frames,
    uint32_t samples_per_packet,
    uint32_t samples_per_second);
```

#### `switch_rtp_deactivate_jitter_buffer()` - Disable Jitter Buffer
```c
SWITCH_DECLARE(switch_status_t) switch_rtp_deactivate_jitter_buffer(switch_rtp_t *rtp_session);
```

### Jitter Calculation

The implementation includes sophisticated jitter measurement:

```c
static void check_jitter(switch_rtp_t *rtp_session) {
  // Calculate RFC3550-compliant jitter measurement
  // J(i) = J(i-1) + (|D(i-1,i)| - J(i-1))/16
}
```

## Statistics and Quality Metrics

### RTP Statistics Structure

Comprehensive statistics are maintained in `switch_rtp_stats_t`:

```c
typedef struct switch_rtp_stats_s {
    switch_rtp_stats_inbound_t inbound;   // Inbound stream stats
    switch_rtp_stats_outbound_t outbound; // Outbound stream stats
} switch_rtp_stats_t;
```

### Key Metrics Tracked

**Inbound Statistics:**
- Packet count and byte count
- Packet loss and late packets
- Jitter measurements (RFC3550 compliant)
- Duplicate packets
- Sequence number gaps
- Codec-specific error rates

**Outbound Statistics:**
- Transmitted packet and byte counts
- RTCP packet transmission
- Bandwidth utilization
- Transmission timestamps

**Quality Metrics:**
- Mean Opinion Score (MOS) estimation
- Round-trip time measurements
- Packet loss percentages
- Jitter variance calculations

### Statistics Functions

#### `switch_rtp_get_stats()` - Get Statistics
```c
SWITCH_DECLARE(switch_rtp_stats_t *) switch_rtp_get_stats(switch_rtp_t *rtp_session,
    switch_memory_pool_t *pool);
```

Returns a complete copy of current RTP statistics.

### MOS (Mean Opinion Score) Calculation

The implementation includes MOS estimation based on:
- Packet loss percentage
- Jitter measurements
- Codec-specific quality factors

```c
static void do_mos(switch_rtp_t *rtp_session) {
  // Calculates estimated MOS score
  // Based on ITU-T G.107 E-model principles
  // Considers packet loss, jitter, and codec factors
}
```

## Debugging Aids and Flags

### Debug Compilation Options

Multiple debugging modes can be enabled at compile time:

```c
//#define DEBUG_TS_ROLLOVER  // Timestamp rollover debugging
//#define DEBUG_2833  // RFC2833 DTMF debugging
//#define RTP_DEBUG_WRITE_DELTA  // Write timing debugging
//#define DEBUG_MISSED_SEQ  // Sequence number debugging
//#define DEBUG_EXTRA  // Extra verbose debugging
//#define DEBUG_RTCP  // RTCP debugging
//#define DEBUG_ESTIMATORS_  // Quality estimator debugging
```

### RTP Flags

The RTP implementation uses extensive flag-based configuration:

**Key Flags:**
- `SWITCH_RTP_FLAG_DTMF_ON`: DTMF processing active
- `SWITCH_RTP_FLAG_PASS_RFC2833`: Pass-through RFC2833 packets
- `SWITCH_RTP_FLAG_AUTO_CNG`: Automatic comfort noise generation
- `SWITCH_RTP_FLAG_SECURE_SEND`: SRTP encryption enabled
- `SWITCH_RTP_FLAG_SECURE_RECV`: SRTP decryption enabled
- `SWITCH_RTP_FLAG_VIDEO`: Video stream mode
- `SWITCH_RTP_FLAG_PROXY_MEDIA`: Media proxy mode
- `SWITCH_RTP_FLAG_RTCP_MUX`: RTCP multiplexing on RTP port

### Debugging Functions

```c
// Enable/disable RTP flags
SWITCH_DECLARE(void) switch_rtp_set_flag(switch_rtp_t *rtp_session,
    switch_rtp_flag_t flag);
SWITCH_DECLARE(void) switch_rtp_clear_flag(switch_rtp_t *rtp_session,
    switch_rtp_flag_t flag);

// Test flag status
SWITCH_DECLARE(uint32_t) switch_rtp_test_flag(switch_rtp_t *rtp_session,
    switch_rtp_flag_t flags);

// Debugging specific functions
SWITCH_DECLARE(switch_status_t) switch_rtp_debug_jitter_buffer(switch_rtp_t *rtp_session,
    const char *name);
```

### Bug Compatibility Flags

The implementation includes flags for handling broken RTP implementations:

```c
typedef enum {
    RTP_BUG_CISCO_SKIP_MARK_BIT_2833    = (1 << 0),
    RTP_BUG_SONUS_SEND_INVALID_TIMESTAMP_2833 = (1 << 1),
    RTP_BUG_IGNORE_MARK_BIT             = (1 << 2),
    RTP_BUG_SEND_LINEAR_TIMESTAMPS      = (1 << 3),
    RTP_BUG_START_SEQ_AT_ZERO           = (1 << 4),
    RTP_BUG_NEVER_SEND_MARKER           = (1 << 5),
    RTP_BUG_IGNORE_DTMF_DURATION        = (1 << 6),
    RTP_BUG_ACCEPT_ANY_RT_EVENT_TYPE    = (1 << 7),
    RTP_BUG_GEN_ONE_GEN_ALL             = (1 << 8),
    RTP_BUG_CHANGE_SSRC_ON_MARKER       = (1 << 9),
    RTP_BUG_FLUSH_JB_ON_DTMF            = (1 << 10)
} switch_rtp_bug_flag_t;
```

## Integration Architecture

**Payload Management**: Dynamic payload mapping with `pmaps`, separate telephony event (`te`) and comfort noise (`cng_pt`) handling

**Timer Integration**: Main media timer plus write timing timer for precise control
**STUN Support**: Built-in NAT traversal with 1-second frequency (`RTP_STUN_FREQ`)

**VAD Integration**: Background noise tracking, hangunder/hangover thresholds, talk time statistics

## Key Files for RTP Issues

**Core**: `switch_core_media.c` (negotiation), `switch_jitterbuffer.c`, `switch_stun.c`, `switch_ice.c`
**Config**: `switch.conf.xml`, `vars.xml` (RTP variables)
**Debug**: Enable `DEBUG_2833`, `DEBUG_MISSED_SEQ` flags, use PCAP analysis

**Common Issues**:
- **Audio Quality**: Check jitter buffer config, codec negotiation
- **DTMF**: Enable RFC2833 debugging, verify payload types
- **Security**: SRTP error counters, DTLS certificate validation
- **NAT**: ICE candidate selection, STUN responses
