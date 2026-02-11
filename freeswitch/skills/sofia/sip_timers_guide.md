# SIP Timers in FreeSWITCH Sofia Implementation

## Overview

This comprehensive guide documents how SIP timers work in FreeSWITCH's Sofia-SIP implementation, covering RFC 3261 timer mechanics, transport-specific behavior, timeout scenarios, DNS integration, and practical configuration.

## 1. SIP Timer Fundamentals in Sofia/FreeSWITCH

### 1.1 RFC 3261 Standard Timers

Sofia-SIP implements the standard RFC 3261 timer values as defined in `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/sofia-sip/nta.h`:

```c
enum {
  NTA_SIP_T1 = 500,      // SIP T1, 500 milliseconds
  NTA_SIP_T2 = 4000,     // SIP T2, 4 seconds
  NTA_SIP_T4 = 5000,     // SIP T4, 5 seconds
  NTA_TIME_MAX = 15 * 24 * 3600 * 1000  // Maximum timer value
};
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/sofia-sip/nta.h` lines 107-114

### 1.2 Timer Hierarchy and Derived Values

The NTA agent structure maintains these timers in `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c`:

```c
struct nta_agent_s {
  unsigned sa_t1;        // Initial retransmit interval (500 ms)
  unsigned sa_t2;        // Maximum retransmit interval (4000 ms)  
  unsigned sa_t4;        // Clear message time (5000 ms)
  unsigned sa_t1x64;     // Transaction lifetime (32 seconds = 64 * T1)
  unsigned sa_timer_c;   // INVITE proxy timeout (185 seconds default)
  unsigned sa_progress;  // Progress timer for provisional responses (60s)
};
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 186-201

**Key Derived Values:**
- **T1x64 (Timer B/F)**: 32 seconds = 64 × 500ms (default transaction timeout)
- **Timer C**: 185 seconds (default proxy INVITE timeout)
- **Timer D**: 32 seconds (INVITE completed state duration)
- **Timer K**: 5 seconds = T4 (non-INVITE completed state duration)

### 1.3 Timer Assignment by Transaction State

Sofia organizes outgoing transactions into queues with associated timers:

```c
// From nta.c initialization
outgoing_queue_init(agent->sa_out.trying, agent->sa_t1x64);        // Timer F
outgoing_queue_init(agent->sa_out.completed, agent->sa_t4);        // Timer K
outgoing_queue_init(agent->sa_out.inv_calling, agent->sa_t1x64);   // Timer B
outgoing_queue_init(agent->sa_out.inv_proceeding, timer_c);        // Timer C
outgoing_queue_init(agent->sa_out.inv_completed, timer_d);         // Timer D
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (agent initialization, around line 1200)

### 1.4 Transaction Timer Mapping

Each `nta_outgoing_t` (client transaction) structure tracks:

```c
struct nta_outgoing_s {
  uint32_t orq_retry;     // Timer A, E (retransmission)
  uint32_t orq_timeout;   // Timer B, D, F, K (transaction timeout)
  unsigned short orq_interval; // Next timer A/E interval
};
```

**Timer Assignments:**
- **Timer A**: Initial INVITE retransmit (starts at T1, doubles up to T2)
- **Timer B**: INVITE transaction timeout (64*T1 = 32 seconds)
- **Timer C**: INVITE proceeding timeout (185 seconds)
- **Timer D**: INVITE wait time after final response (32 seconds)
- **Timer E**: Non-INVITE retransmit (starts at T1, doubles up to T2)
- **Timer F**: Non-INVITE transaction timeout (64*T1 = 32 seconds)
- **Timer K**: Non-INVITE wait time after final response (T4 = 5 seconds)

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 450-530 (outgoing transaction structure)

## 2. Protocol-Specific Behavior

### 2.1 Transport Reliability Detection

Sofia-SIP checks transport reliability using `tport_is_reliable()` which determines whether retransmissions are needed:

```c
orq->orq_reliable = tport_is_reliable(tp);

if (!orq->orq_reliable) {
  // Enable retransmission timers for unreliable transports
  unsigned t1_timer = agent->sa_t1;
  if (t1_timer < 1000) t1_timer = 1000;  // Minimum 1 second
  outgoing_set_timer(orq, t1_timer);     // Start Timer A/E
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (outgoing_send function, around line 5800)

### 2.2 UDP Transport (Unreliable)

**Characteristics:**
- **Retransmissions**: Enabled (Timer A/E)
- **Exponential Backoff**: T1 → 2*T1 → 4*T1 → ... → T2 (max)
- **Transaction Timeout**: Timer B (INVITE) or Timer F (non-INVITE) at 64*T1
- **Packet Loss Handling**: Automatic retransmits every interval until timeout

**Retransmission Schedule for INVITE (UDP):**
```
Time 0ms:     Send INVITE
Time 500ms:   Retransmit (T1)
Time 1500ms:  Retransmit (2*T1)
Time 3500ms:  Retransmit (4*T1)
Time 7500ms:  Retransmit (T2 = 4000ms max interval)
Time 11500ms: Retransmit (T2)
Time 15500ms: Retransmit (T2)
...continues until 32000ms (Timer B fires)
```

**Code Evidence:**
The retransmission interval doubles until reaching T2:
```c
// Timer A/E exponential backoff
if (orq->orq_interval < agent->sa_t2) {
  orq->orq_interval *= 2;
  if (orq->orq_interval > agent->sa_t2)
    orq->orq_interval = agent->sa_t2;
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (outgoing retransmit logic)

### 2.3 TCP Transport (Reliable)

**Characteristics:**
- **Retransmissions**: Disabled (no Timer A/E)
- **Transaction Timeout**: Still active (Timer B/F)
- **Connection-Oriented**: Single send, TCP handles reliability
- **Timer N3**: Special 5-second timer (T4) for connection establishment

**Key Difference:**
```c
if (orq->orq_reliable) {
  // No retransmission timers set
  // Only transaction timeout (Timer B/F) is active
}
```

**TCP Connection Timeout (Timer N3):**
If TCP connection is not established within T4 (5 seconds):
```c
else if (orq->orq_try_tcp_instead && !tport_is_connected(tp)) {
  outgoing_set_timer(orq, agent->sa_t4); // Timer N3
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 5800-5850

### 2.4 TLS Transport (Reliable + Encrypted)

**Characteristics:**
- **Retransmissions**: Disabled (like TCP)
- **Connection Timeout**: Configurable TLS-specific timeout
- **Handshake Overhead**: Additional time for TLS negotiation
- **Fast Fail**: Optional per-call TLS connect failure behavior

**TLS-Specific Timer:**
```c
unsigned tls_reconnect_interval = (orq->orq_call_tls_connect_timeout_is_set) ? 
    orq->orq_call_tls_connect_timeout : agent->sa_tls_orq_connect_timeout;

if (su_casenmatch(orq->orq_tpn->tpn_proto, "tls", 3) && !tport_is_connected(tp)) {
  outgoing_set_timer(orq, tls_reconnect_interval);
}
```

**Configuration:**
- `NTATAG_TLS_ORQ_CONNECT_TIMEOUT`: Global TLS connection timeout
- Per-call override: `orq_call_tls_connect_timeout`
- `NTATAG_TLS_ORQ_CONNECT_FAIL_FAST`: Fail immediately on TLS errors

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (TLS connection handling)

### 2.5 WebSocket Transport

**Characteristics:**
- **Retransmissions**: Disabled (reliable like TCP)
- **Frame-Based**: Messages sent as WebSocket frames
- **Timer Behavior**: Same as TCP (no retransmits, only transaction timeouts)

**Transport Detection:**
```c
agent->sa_tport_ws  = 1;  // WebSocket support
agent->sa_tport_wss = 1;  // Secure WebSocket support
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 237-244 (agent flags)

### 2.6 Transport Failover Logic

Sofia implements automatic transport failover on failures:

**UDP to TCP on Message Size:**
```c
if (err == EMSGSIZE && !orq->orq_try_tcp_instead) {
  if (su_casematch(tpn->tpn_proto, "udp") || su_casematch(tpn->tpn_proto, "*")) {
    outgoing_try_tcp_instead(orq);  // Switch to TCP
    continue;
  }
}
```

**TCP to UDP on Connection Refused:**
```c
else if (err == ECONNREFUSED && orq->orq_try_tcp_instead) {
  if (su_casematch(tpn->tpn_proto, "tcp") && msg_size(msg) <= 65535) {
    outgoing_try_udp_instead(orq, 0);  // Fallback to UDP
    continue;
  }
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (outgoing_prepare_send function)

## 3. Timeout Behavior and Call Flows

### 3.1 Timer A - INVITE Retransmission (UDP only)

**Purpose**: Retransmit INVITE requests on unreliable transports

**Behavior:**
- Initial interval: T1 (500ms)
- Doubles each time: T1 → 2T1 → 4T1 → T2
- Maximum interval: T2 (4000ms)
- Stops when: Any provisional/final response received OR Timer B fires

**Code Implementation:**
```c
// From _nta_outgoing_timer()
while ((orq = rq->q_head)) {
  if ((int32_t)(orq->orq_retry - now) > 0)
    break;
  
  outgoing_retransmit(orq);  // Resend
  
  // Double interval up to T2
  if (orq->orq_interval < agent->sa_t2)
    orq->orq_interval *= 2;
  
  outgoing_set_timer(orq, orq->orq_interval);
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (_nta_outgoing_timer function)

### 3.2 Timer B - INVITE Transaction Timeout

**Purpose**: Limit total time waiting for INVITE response

**Behavior:**
- Duration: 64*T1 = 32 seconds (default)
- Fires when: No final response received within timeout
- Action: Transaction fails with timeout error
- Applies to: Both reliable and unreliable transports

**Call Flow - Timer B Timeout:**
```
UAC                          UAS
 |                            |
 |-----INVITE (0ms)---------->| (message lost)
 |-----INVITE (500ms)-------->| (message lost)
 |-----INVITE (1500ms)------->| (message lost)
 |-----INVITE (3500ms)------->| (message lost)
 ...retransmits continue...
 |                            |
 |<===TIMEOUT (32000ms)===    |
 |                            |
 +-- 408 Request Timeout      |
     generated locally         |
```

**Code Implementation:**
```c
static size_t outgoing_timer_bf(outgoing_queue_t *q, char const *timer, uint32_t now) {
  while ((orq = q->q_head)) {
    if ((int32_t)(orq->orq_timeout - now) > 0)
      break;
    
    outgoing_timeout(orq, now);  // Transaction timeout
    timeout++;
  }
}

static void outgoing_timeout(nta_outgoing_t *orq, uint32_t now) {
  if (agent->sa_timeout_408) {
    // Generate local 408 Request Timeout
    outgoing_reply(orq, SIP_408_REQUEST_TIMEOUT, 0);
  } else {
    outgoing_terminate(orq);
  }
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (timer B/F handling)

### 3.3 Timer C - INVITE Proceeding Timeout

**Purpose**: Limit time waiting in INVITE proceeding state (after receiving 1xx)

**Behavior:**
- Duration: 185 seconds (default) or 3 minutes+ typical
- Starts when: First 1xx provisional response received
- Fires when: No final response after extended period
- Action: Proxy cancels transaction

**Note**: Timer C is primarily for proxies. UAs typically don't implement it unless configured.

**Configuration Check:**
```c
if (agent->sa_use_timer_c || !agent->sa_is_a_uas)
  timer_c = agent->sa_timer_c;  // 185000ms default
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 1180-1195

### 3.4 Timer D - INVITE Completed State Wait

**Purpose**: Absorb retransmissions of final response in completed state

**Behavior:**
- Duration: 32 seconds (max of T1x64 or 32s)
- Starts when: Final response (2xx-6xx) sent/received for INVITE
- Purpose: Catch retransmitted final responses (UDP)
- On reliable transports: Often shortened to 0

**Call Flow:**
```
UAC                          UAS
 |                            |
 |-------INVITE-------------->|
 |<------200 OK (0ms)---------|
 |                            |
 +-- Enter COMPLETED state    |
 |   (Timer D = 32s)          |
 |                            |
 |<------200 OK (500ms)-------|  (retransmit)
 |   (absorbed, no callback)  |
 |<------200 OK (1500ms)------|  (retransmit)
 |   (absorbed, no callback)  |
 |                            |
 +-- Timer D expires (32s)    |
 +-- Transaction destroyed    |
```

**Code:**
```c
if (orq->orq_method == sip_method_invite) {
  outgoing_queue(orq->orq_agent->sa_out.inv_completed, orq); // Timer D
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (outgoing completion handling)

### 3.5 Timer E - Non-INVITE Retransmission (UDP only)

**Purpose**: Retransmit non-INVITE requests (OPTIONS, REGISTER, etc.)

**Behavior:**
- Same as Timer A: T1 → 2T1 → 4T1 → T2
- Initial: 500ms
- Maximum interval: T2 (4000ms)
- Total duration: Limited by Timer F (32 seconds)

**Difference from Timer A:**
- Timer A: INVITE only
- Timer E: All other methods (REGISTER, OPTIONS, BYE, etc.)
- Both follow identical exponential backoff pattern

### 3.6 Timer F - Non-INVITE Transaction Timeout

**Purpose**: Limit total time waiting for non-INVITE response

**Behavior:**
- Duration: 64*T1 = 32 seconds (default)
- Applies to: REGISTER, OPTIONS, SUBSCRIBE, NOTIFY, etc.
- Action on timeout: 408 generated locally
- Both UDP and TCP: Timer F always active

**Call Flow - Timer F:**
```
UAC                          UAS
 |                            |
 |-------REGISTER (0ms)------>| (lost)
 |-------REGISTER (500ms)---->| (lost)
 |-------REGISTER (1500ms)--->| (lost)
 ...retransmits continue...
 |                            |
 |<===TIMEOUT (32000ms)===    |
 |                            |
 +-- 408 Request Timeout      |
```

### 3.7 Timer K - Non-INVITE Completed State Wait

**Purpose**: Absorb retransmissions of final non-INVITE responses

**Behavior:**
- Duration: T4 = 5 seconds
- Much shorter than Timer D (32s)
- Reason: Non-INVITE responses are smaller, less likely delayed

**Code:**
```c
outgoing_queue_init(agent->sa_out.completed, agent->sa_t4); // Timer K
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (queue initialization)

### 3.8 Timer N3 - TCP Connection Establishment

**Purpose**: Timeout for TCP connection attempts

**Behavior:**
- Duration: T4 = 5 seconds
- Applies when: TCP connection not yet established
- On timeout: Try UDP fallback OR try alternate destination

**Implementation:**
```c
else if (orq->orq_try_tcp_instead && !tport_is_connected(tp)) {
  outgoing_set_timer(orq, agent->sa_t4); // Timer N3
}

// On Timer N3 expiry:
if (orq->orq_try_tcp_instead && !tport_is_connected(tp)) {
  // Timer N3: try to use UDP if trying to send via TCP
  // but no connection is established within SIP T4
  outgoing_try_udp_instead(orq, 1);
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (Timer N3 logic)

### 3.9 Server Transaction Timers (Incoming)

**Timer G**: Retransmit INVITE final response (UDP)
```c
struct nta_incoming_s {
  uint32_t irq_timeout;    // Timer H, I, J
  uint32_t irq_retry;      // Timer G
  unsigned irq_reliable_tp:1;  // Transport is reliable
};
```

**Timer H**: Wait for ACK to INVITE final response
- Duration: 64*T1 = 32 seconds
- Purpose: Ensure ACK is received for INVITE 3xx-6xx responses
- On timeout: Destroy transaction

**Timer I**: Wait time in confirmed state
- Duration: T4 = 5 seconds (for unreliable), 0 (for reliable)
- Purpose: Absorb retransmitted ACKs

**Timer J**: Wait time for non-INVITE transaction
- Duration: 64*T1 = 32 seconds (unreliable) or 0 (reliable)

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (incoming transaction structure)

## 4. DNS and SRV Integration

### 4.1 DNS Resolver Architecture

Sofia-SIP includes an integrated DNS resolver (`sres_resolver_t`) that interacts with SIP timers:

```c
struct nta_agent_s {
  sres_resolver_t *sa_resolver;      // DNS resolver
  enum nta_res_order_e sa_res_order; // AAAA/A order
  unsigned sa_use_naptr : 1;         // Use NAPTR lookup
  unsigned sa_use_srv : 1;           // Use SRV lookup
  unsigned sa_srv_503 : 1;           // Try alternate on 503 (RFC 3263)
};
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 172-176, 246-249

### 4.2 DNS Resolution Timeout

**DNS-specific timeouts** (from `sres.c`):
```c
#define SRES_RETRY_INTERVAL    // DNS query retry interval
#define DNS_ICMP_TIMEOUT 60    // 60 seconds
#define DNS_ERROR_TIMEOUT 10   // 10 seconds

// Query timeout handling
if (q->q_retry_count) {
  status = SRES_TIMEOUT_ERR;
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/sresolv/sres.c`

### 4.3 SRV Record Failover with Timers

**RFC 3263 Behavior:**
- Sofia maintains list of resolved SRV targets
- On timeout or 503 response, tries next SRV target
- SRV failover interacts with SIP transaction timers

**Transaction State During DNS Resolution:**
```c
outgoing_queue_t resolving[1];  // Transactions in DNS resolution

// Outgoing transaction states
typedef struct sipdns_resolver sipdns_resolver_t;
struct nta_outgoing_s {
  sipdns_resolver_t *orq_resolver;
};
```

**Call Flow with SRV Failover:**
```
UAC                     DNS             SRV Target 1      SRV Target 2
 |                       |                    |                |
 |--DNS SRV query------->|                    |                |
 |<-SRV records----------|                    |                |
 |  (priority 10: srv1)                       |                |
 |  (priority 20: srv2)                       |                |
 |                                            |                |
 |--------INVITE (0ms)----------------------->|                |
 |--------INVITE (500ms)--------------------->| (no response)  |
 |--------INVITE (1500ms)-------------------->| (timeout)      |
 ...Timer B counting down...                  |                |
 |                                            |                |
 |<==Trying alternate (Timer N3 or error)==   |                |
 |                                            |                |
 |--------INVITE (retry to srv2)----------------------------->|
 |<-------100 Trying--------------------------------------------|
 |<-------200 OK-----------------------------------------------|
```

**Code Implementation:**
```c
// On timeout with alternate destinations
if (outgoing_other_destinations(orq)) {
  SU_DEBUG_5(("nta: timer %s fired, trying alternative server for %s\n",
              "N3", orq->orq_method_name));
  outgoing_try_another(orq);
} else {
  outgoing_timeout(orq, now);
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (_nta_outgoing_timer function)

### 4.4 NAPTR/SRV/A Record Lookup Sequence

**Lookup Order (RFC 3263):**
1. NAPTR records (if `sa_use_naptr` enabled)
2. SRV records (if `sa_use_srv` enabled)
3. A/AAAA records (based on `sa_res_order`)

**Resolution Order Configuration:**
```c
enum nta_res_order_e {
  nta_res_ip6_ip4,   // Try IPv6 first, then IPv4
  nta_res_ip4_ip6,   // Try IPv4 first, then IPv6
  nta_res_ip6_only,  // IPv6 only
  nta_res_ip4_only   // IPv4 only
};
```

**Impact on Timers:**
- DNS queries are sequential (NAPTR → SRV → A)
- Each DNS query can timeout independently
- Total resolution time can delay transaction start
- Timer B/F doesn't start until DNS completes OR fails

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 116-123

### 4.5 DNS Failure Handling

**On DNS Timeout:**
```c
int outgoing_resolving_error(nta_outgoing_t *orq, int status, char const *phrase) {
  if (status == 503 || status >= 500) {
    // Try next destination if available
    if (outgoing_other_destinations(orq))
      return outgoing_resolving(orq);
  }
  
  // Report error to application
  outgoing_reply(orq, status, phrase, 0);
}
```

**Error Codes:**
- `SIPDNS_503_ERROR`: DNS resolution failed (503 Service Unavailable)
- `SIP_500_INTERNAL_SERVER_ERROR`: Internal DNS error

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (outgoing_resolving_error function)

## 5. Sofia-SIP Implementation Details

### 5.1 Timer Management Architecture

**Central Timer Function:**
```c
static void agent_timer(nta_agent_t *agent, su_timer_t *timer, su_timer_arg_t *arg) {
  uint32_t now = su_time_ms(su_now());
  
  agent->sa_in_timer = 1;
  
  _nta_outgoing_timer(agent);  // Process outgoing transaction timers
  _nta_incoming_timer(agent);  // Process incoming transaction timers
  
  agent->sa_in_timer = 0;
}
```

**Key Data Structures:**
- `su_timer_t *sa_timer`: Sofia-SU timer object
- `uint32_t sa_next`: Timestamp for next timer firing
- Timer resolution: Milliseconds (uint32_t wraparound after 49.7 days)

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (agent_timer function)

### 5.2 Outgoing Transaction Timer Queues

**Queue-Based Timer Management:**
Sofia maintains multiple queues for outgoing transactions, each with specific timeouts:

```c
typedef struct outgoing_queue_t {
  nta_outgoing_t **q_tail;
  nta_outgoing_t  *q_head;
  size_t           q_length;
  unsigned         q_timeout;  // Associated timer duration
} outgoing_queue_t;
```

**Queue Organization:**
```c
struct {
  outgoing_queue_t  *re_list;        // Retransmission list (Timer A/E)
  outgoing_queue_t  *re_t1;          // T1 shortcut pointer
  size_t             re_length;      // Retransmit queue length
  
  outgoing_queue_t   resolving[1];   // DNS resolution
  outgoing_queue_t   trying[1];      // Timer F (non-INVITE)
  outgoing_queue_t   completed[1];   // Timer K
  outgoing_queue_t   inv_calling[1]; // Timer B (INVITE)
  outgoing_queue_t   inv_proceeding[1]; // Timer C
  outgoing_queue_t   inv_completed[1];  // Timer D
} sa_out;
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 129-141

### 5.3 Retransmission List Optimization

**Fast Timer Lookup:**
Sofia maintains a separate retransmission list (`re_list`) with a shortcut pointer (`re_t1`) to optimize timer checks:

```c
static void outgoing_set_timer(nta_outgoing_t *orq, uint32_t interval) {
  nta_outgoing_t **rq;
  
  orq->orq_retry = set_timeout(orq->orq_agent, orq->orq_interval = interval);
  
  // Shortcut into queue at SIP T1
  rq = orq->orq_agent->sa_out.re_t1;
  
  if (!(*rq) || (int32_t)((*rq)->orq_retry - orq->orq_retry) > 0)
    rq = &orq->orq_agent->sa_out.re_list;
  
  // Insert sorted by retry time
  while (*rq && (int32_t)((*rq)->orq_retry - orq->orq_retry) <= 0)
    rq = &(*rq)->orq_rprev;
  
  orq->orq_rprev = rq;
}
```

**Optimization**: The `re_t1` pointer provides O(1) access to transactions near T1 interval, avoiding full list traversal.

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (outgoing_set_timer function)

### 5.4 Timer Processing Loop

**Efficient Timer Firing:**
```c
static void _nta_outgoing_timer(nta_agent_t *sa) {
  uint32_t now = su_time_ms(su_now());
  
  // Process retransmission list (Timer A/E)
  while ((orq = sa->sa_out.re_list)) {
    if ((int32_t)(orq->orq_retry - now) > 0)
      break;  // No more expired timers
    
    outgoing_retransmit(orq);
    
    // Exponential backoff
    if (orq->orq_interval < sa->sa_t2)
      orq->orq_interval *= 2;
    
    outgoing_set_timer(orq, orq->orq_interval);
  }
  
  // Process timeout queues (Timer B/F/D/K/C)
  timeout += outgoing_timer_bf(sa->sa_out.inv_calling, "B", now);
  timeout += outgoing_timer_bf(sa->sa_out.trying, "F", now);
  timeout += outgoing_timer_dk(sa->sa_out.inv_completed, "D", now);
  timeout += outgoing_timer_dk(sa->sa_out.completed, "K", now);
  timeout += outgoing_timer_c(sa->sa_out.inv_proceeding, "C", now);
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (_nta_outgoing_timer function)

### 5.5 Timer Calculation and Wraparound Handling

**Safe Timer Arithmetic:**
Sofia uses signed 32-bit arithmetic to handle wraparound:

```c
// Check if timer expired (handles uint32_t wraparound)
if ((int32_t)(orq->orq_retry - now) > 0)
  break;  // Not expired yet

// Set timeout relative to now
uint32_t set_timeout(nta_agent_t *agent, uint32_t interval) {
  return su_time_ms(su_now()) + interval;
}
```

**Wraparound Safety**: Cast to `int32_t` ensures correct comparison even if timestamp wraps at UINT32_MAX.

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c`

### 5.6 Minimum T1 Enforcement

**Race Condition Mitigation:**
Sofia enforces a minimum T1 of 1000ms to avoid race conditions:

```c
if (!orq->orq_reliable) {
  // Race condition on initial t1 timer timeout
  // Set minimum initial timeout to 1000ms
  unsigned t1_timer = agent->sa_t1;
  if (t1_timer < 1000) t1_timer = 1000;
  outgoing_set_timer(orq, t1_timer);
}
```

**Reason**: Prevents pathological behavior if T1 is set too low (< 500ms).

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 5800-5810

### 5.7 Maximum Retransmissions

**Server Transaction Limit:**
```c
enum {
  timer_max_retransmit = 30  // Maximum retransmissions
};

// In _nta_incoming_timer()
if (retransmitted >= timer_max_retransmit)
  break;  // Stop retransmitting
```

**Purpose**: Prevent infinite retransmissions even if timers misconfigured.

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (_nta_incoming_timer function)

## 6. Practical Configuration

### 6.1 FreeSWITCH Sofia Profile Timer Parameters

**Configuration Location:** `/etc/freeswitch/sip_profiles/*.xml`

**Available Parameters:**
```xml
<profile name="internal">
  <settings>
    <!-- Basic SIP timers -->
    <param name="timer-T1" value="500"/>      <!-- Initial retransmit (ms) -->
    <param name="timer-T1X64" value="32000"/> <!-- Transaction lifetime (ms) -->
    <param name="timer-T2" value="4000"/>     <!-- Max retransmit interval (ms) -->
    <param name="timer-T4" value="4000"/>     <!-- Completed state duration (ms) -->
    
    <!-- Session timers (RFC 4028) -->
    <param name="enable-timer" value="false"/>  <!-- Disable Session-Expires -->
    <param name="session-timeout" value="1800"/> <!-- Session timeout (seconds) -->
    <param name="minimum-session-expires" value="120"/> <!-- Min SE value -->
  </settings>
</profile>
```

**Source Code Reference:**
```c
// From mod_sofia/sofia.c configuration parsing
if (!strcasecmp(var, "timer-T1") && !zstr(val)) {
  int v = atoi(val);
  if (v >= 100 && v <= 10000) {
    profile->timer_t1 = v;
  } else {
    profile->timer_t1 = 500;  // Default
  }
}
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/freeswitch/src/mod/endpoints/mod_sofia/sofia.c` (configuration parser, lines with timer-T1/T2/T4)

### 6.2 NUA-Level Timer Configuration Tags

**Runtime Configuration via NTATAG:**
```c
nua_set_params(profile->nua,
  NTATAG_SIP_T1(profile->timer_t1),        // T1 value
  NTATAG_SIP_T1X64(profile->timer_t1x64),  // Transaction lifetime
  NTATAG_SIP_T2(profile->timer_t2),        // T2 value
  NTATAG_SIP_T4(profile->timer_t4),        // T4 value
  NTATAG_TIMER_C(185000),                  // Timer C (proxy)
  NTATAG_TIMEOUT_408(1),                   // Generate 408 on timeout
  NTATAG_TLS_ORQ_CONNECT_TIMEOUT(5000),    // TLS connect timeout
  TAG_END());
```

**Available NTATAGs** (from `nta_tag.h`):
- `NTATAG_SIP_T1(x)`: Set T1 in milliseconds
- `NTATAG_SIP_T1X64(x)`: Set transaction lifetime (usually 64*T1)
- `NTATAG_SIP_T2(x)`: Set T2 (max retransmit interval)
- `NTATAG_SIP_T4(x)`: Set T4 (completed state duration)
- `NTATAG_TIMER_C(x)`: Set Timer C (INVITE proceeding timeout)
- `NTATAG_TIMEOUT_408(x)`: Generate local 408 on timeout (boolean)
- `NTATAG_PASS_408(x)`: Pass 408 to application (boolean)
- `NTATAG_TLS_ORQ_CONNECT_TIMEOUT(x)`: TLS connection timeout

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/sofia-sip/nta_tag.h`

### 6.3 Profile Structure Timer Fields

**mod_sofia Profile Definition:**
```c
typedef struct sofia_profile_s {
  uint32_t timer_t1;      // SIP T1 timer value
  uint32_t timer_t1x64;   // Transaction lifetime (64*T1)
  uint32_t timer_t2;      // Max retransmit interval
  uint32_t timer_t4;      // Completed state duration
  
  uint32_t session_timeout;           // Session timer (RFC 4028)
  uint32_t minimum_session_expires;   // Minimum Session-Expires
  
  char *timer_name;  // RTP timer name
} sofia_profile_t;
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/freeswitch/src/mod/endpoints/mod_sofia/mod_sofia.h`

### 6.4 Tuning for Different Network Conditions

**Low-Latency Local Network:**
```xml
<param name="timer-T1" value="200"/>      <!-- Faster initial retry -->
<param name="timer-T1X64" value="12800"/> <!-- Shorter transaction lifetime -->
<param name="timer-T2" value="2000"/>     <!-- Lower max interval -->
```

**High-Latency WAN/Satellite:**
```xml
<param name="timer-T1" value="2000"/>     <!-- Longer initial retry -->
<param name="timer-T1X64" value="128000"/> <!-- Extended lifetime (128s) -->
<param name="timer-T2" value="8000"/>     <!-- Higher max interval -->
```

**Mobile/Unreliable Networks:**
```xml
<param name="timer-T1" value="1000"/>     <!-- Conservative initial retry -->
<param name="timer-T1X64" value="64000"/> <!-- Standard lifetime -->
<param name="timer-T2" value="4000"/>     <!-- Standard max interval -->
<!-- Use UDP for better behavior on packet loss -->
```

**Enterprise TCP/TLS Only:**
```xml
<param name="timer-T1" value="500"/>      <!-- Standard (not used for TCP) -->
<param name="timer-T1X64" value="32000"/> <!-- Transaction timeout still active -->
<!-- Retransmissions disabled on TCP/TLS -->
```

### 6.5 Per-Call Timer Overrides

**Application-Level Control:**
FreeSWITCH applications can't directly override timers per-call (timer values are profile-level). However, transport selection affects behavior:

```c
// Force UDP (enables retransmissions)
switch_channel_set_variable(channel, "sip_transport", "udp");

// Force TCP (disables retransmissions)
switch_channel_set_variable(channel, "sip_transport", "tcp");

// TLS with custom timeout (not directly exposed, uses profile default)
switch_channel_set_variable(channel, "sip_transport", "tls");
```

### 6.6 Common Issues and Debugging

#### Issue 1: Premature Timeouts on Slow Networks

**Symptom**: 408 Request Timeout before remote side responds

**Cause**: Default T1x64 (32s) too short for high-latency network

**Solution:**
```xml
<param name="timer-T1X64" value="64000"/>  <!-- 64 seconds -->
```

**Debug:**
```
sofia global siptrace on
sofia loglevel all 9
```

Look for:
```
nta: timer B fired, timeout for INVITE
outgoing_timeout: generating 408 Request Timeout
```

#### Issue 2: Excessive Retransmissions

**Symptom**: Many duplicate INVITEs on wire, bandwidth waste

**Cause**: T1 too low causing aggressive retransmits

**Solution:**
```xml
<param name="timer-T1" value="1000"/>  <!-- More conservative -->
<param name="timer-T2" value="4000"/>  <!-- Standard max -->
```

**Debug:** Check packet captures for retransmit timing:
```bash
tshark -r capture.pcap -Y "sip.Method == INVITE" -T fields -e frame.time_relative -e sip.CSeq
```

#### Issue 3: Slow DNS Causing Transaction Failures

**Symptom**: Calls fail with 408 but network is functional

**Cause**: DNS resolution taking too long, consuming Timer B duration

**Solution:**
1. Optimize DNS (reduce timeout, use local cache)
2. Increase T1x64 to accommodate DNS delay
3. Use IP addresses instead of hostnames for time-critical calls

**Debug:**
```
sofia profile <name> siptrace on
```
Look for:
```
nta: resolving <domain>
(long delay)
nta: timer B fired before resolution complete
```

#### Issue 4: TCP Connection Timeouts

**Symptom**: Calls to TCP endpoints fail even though server is up

**Cause**: Timer N3 (5s) expires before TCP handshake + TLS completes

**Solution:**
```c
// Increase T4 which is used for Timer N3
<param name="timer-T4" value="10000"/>  <!-- 10 seconds -->
```

Or configure TLS-specific timeout (code-level):
```c
NTATAG_TLS_ORQ_CONNECT_TIMEOUT(15000)  // 15 seconds
```

#### Issue 5: Transactions Stuck in Completed State

**Symptom**: Memory growth, transactions not cleaning up

**Cause**: Timer D/K not firing due to timer processing issues

**Debug:**
```
sofia status profile <name>
```
Check for high transaction counts.

**Solution**: Restart profile or investigate timer thread issues.

### 6.7 Monitoring and Metrics

**FreeSWITCH Commands:**
```
# View profile status including transaction counts
sofia status profile internal

# View specific call transaction state
sofia global calls

# Enable detailed SIP trace
sofia global siptrace on
sofia loglevel all 9

# Check for timer-related log messages
grep -i "timer.*fired\|timeout\|retransmit" /var/log/freeswitch/freeswitch.log
```

**Sofia Statistics (via `nta_agent_get_stats`):**
```c
struct {
  usize_t as_recv_msg;
  usize_t as_recv_request;
  usize_t as_recv_response;
  usize_t as_client_tr;      // Active client transactions
  usize_t as_server_tr;      // Active server transactions
  usize_t as_canceled_tr;    // Canceled transactions
} agent_stats;
```

**Key Metrics:**
- High `as_client_tr`: Many pending outgoing transactions (possible timeout issues)
- High `as_canceled_tr`: Frequent cancellations (may indicate Timer C timeouts)
- Increasing transaction counts: Possible timer processing issues

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` lines 282-301 (statistics structure)

## 7. Advanced Topics

### 7.1 Timer Interaction with 100rel (PRACK)

**100rel Extension** introduces additional timers:

```c
// Timer P1: PRACK retransmission timer
// Timer P2: PRACK transaction timeout (similar to Timer F)
```

**Code References:**
```c
/* Timer P1 - PRACK timer */
outgoing_set_timer(orq, agent->sa_t1);

/* Timer P2 - PRACK timer */
outgoing_set_timer(orq, agent->sa_t1x64);
```

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (reliable provisional response handling)

### 7.2 Timer Behavior with SigComp

**Compression Impact:**
- Timer values unchanged
- Retransmissions compress better (dictionary built up)
- Decompression failures may trigger immediate retransmit

### 7.3 Timer C for UAS vs Proxy

**User Agent Server (UAS):**
```c
if (!agent->sa_is_a_uas)
  timer_c = agent->sa_timer_c;  // Enable Timer C
else
  timer_c = 0;  // Disable for pure UAS
```

FreeSWITCH typically operates as B2BUA, so Timer C behavior depends on `sa_use_timer_c` flag.

### 7.4 Impact of Packet Loss Simulation

**Testing Mode:**
```c
if (agent->sa_drop_prob && !tport_is_reliable(tport)) {
  if ((unsigned)su_randint(0, 1000) < agent->sa_drop_prob) {
    SU_DEBUG_5(("nta: dropped simulating packet loss\n"));
    return;  // Drop packet
  }
}
```

**Configuration:**
```c
NTATAG_DROP_PROB(100)  // Drop 10% of packets (value is per-mille: 0-1000)
```

**Effect**: Triggers retransmission timers even on reliable networks for testing.

**Source**: `/home/jerome.poulin/GIT/freeswitch/sofia-sip/libsofia-sip-ua/nta/nta.c` (packet drop simulation)

## 8. Summary and Quick Reference

### 8.1 Timer Quick Reference Table

| Timer | Purpose | Duration | Transport | RFC Section |
|-------|---------|----------|-----------|-------------|
| T1 | Base timer | 500ms | All | RFC 3261 17.1.1.1 |
| T2 | Max retransmit interval | 4000ms | All | RFC 3261 17.1.1.1 |
| T4 | Completed wait / Max msg lifetime | 5000ms | All | RFC 3261 17.1.1.1 |
| A | INVITE retransmit | T1, 2T1, 4T1...T2 | UDP only | RFC 3261 17.1.1.2 |
| B | INVITE timeout | 64*T1 (32s) | All | RFC 3261 17.1.1.2 |
| C | INVITE proceeding | 185s (3+ min) | All | RFC 3261 16.6 |
| D | INVITE completed wait | 32s | All | RFC 3261 17.1.1.2 |
| E | Non-INVITE retransmit | T1, 2T1, 4T1...T2 | UDP only | RFC 3261 17.1.2.2 |
| F | Non-INVITE timeout | 64*T1 (32s) | All | RFC 3261 17.1.2.2 |
| G | Response retransmit | T1, 2T1...T2 | UDP only | RFC 3261 17.2.1 |
| H | ACK wait | 64*T1 (32s) | All | RFC 3261 17.2.1 |
| I | ACK confirmed wait | T4 (5s) | UDP, 0 for TCP | RFC 3261 17.2.1 |
| J | Non-INVITE completed | 64*T1 (32s) | UDP, 0 for TCP | RFC 3261 17.2.2 |
| K | Non-INVITE completed | T4 (5s) | All | RFC 3261 17.1.2.2 |
| N3 | TCP connection | T4 (5s) | TCP/TLS | Implementation |

### 8.2 Key Implementation Files

| File | Lines | Description |
|------|-------|-------------|
| `sofia-sip/libsofia-sip-ua/nta/nta.c` | ~17000 | Main NTA implementation with all timer logic |
| `sofia-sip/libsofia-sip-ua/nta/sofia-sip/nta.h` | ~500 | NTA public API and timer constants |
| `sofia-sip/libsofia-sip-ua/nta/sofia-sip/nta_tag.h` | ~400 | NTATAG parameter definitions |
| `freeswitch/src/mod/endpoints/mod_sofia/sofia.c` | ~6000 | FreeSWITCH Sofia configuration parsing |
| `freeswitch/src/mod/endpoints/mod_sofia/mod_sofia.h` | ~1500 | Sofia profile structure definitions |
| `sofia-sip/libsofia-sip-ua/sresolv/sres.c` | ~5000 | DNS resolver with timeout handling |

### 8.3 Critical Code Functions

**Timer Management:**
- `agent_timer()`: Main timer callback
- `_nta_outgoing_timer()`: Process all outgoing transaction timers
- `_nta_incoming_timer()`: Process all incoming transaction timers
- `outgoing_set_timer()`: Set retransmission timer (A/E)
- `outgoing_timeout()`: Handle transaction timeout (B/F)
- `outgoing_timer_bf()`: Process Timer B/F expiry
- `outgoing_timer_dk()`: Process Timer D/K expiry
- `outgoing_timer_c()`: Process Timer C expiry

**Transport Handling:**
- `tport_is_reliable()`: Check if transport needs retransmissions
- `outgoing_try_tcp_instead()`: Failover UDP→TCP on EMSGSIZE
- `outgoing_try_udp_instead()`: Failover TCP→UDP on ECONNREFUSED

**DNS Integration:**
- `outgoing_resolve()`: Initiate DNS resolution
- `outgoing_resolving()`: DNS resolution in progress
- `outgoing_resolving_error()`: DNS failure handling
- `outgoing_other_destinations()`: Check for SRV failover targets

### 8.4 Configuration Best Practices

**General Recommendations:**
1. **Use defaults** unless you have specific network requirements
2. **Match T1 to network latency**: Low latency = lower T1, high latency = higher T1
3. **T1X64 should be ≥ DNS timeout + network latency**
4. **T2 should be ≥ 2*RTT** to avoid premature timeouts
5. **Don't set T1 < 200ms** (except in controlled lab environments)
6. **For TCP/TLS, increase T4** if connections are slow to establish
7. **Monitor transaction counts** to detect timer issues early

**Validation:**
```xml
<!-- Ensure these relationships hold -->
T1 ≥ 100ms (Sofia minimum)
T2 ≥ T1
T4 ≥ T1
T1X64 = 64 * T1 (typically)
Timer C ≥ T1X64 (for proxies)
```

### 8.5 Troubleshooting Checklist

**Call Failures with 408:**
- [ ] Check T1X64 value (sufficient for network RTT?)
- [ ] Verify DNS resolution time (may consume transaction timeout)
- [ ] Check remote endpoint is responding (packet capture)
- [ ] Review sofia loglevel 9 logs for timer messages
- [ ] Test with IP address (eliminate DNS)

**Excessive Retransmissions:**
- [ ] Verify T1 not set too low (< 500ms)
- [ ] Check T2 appropriate for network
- [ ] Confirm UDP transport (TCP should not retransmit)
- [ ] Look for network packet loss (causes legitimate retransmits)

**Slow Call Setup:**
- [ ] Check DNS resolution time
- [ ] Verify SRV record failover not occurring unnecessarily
- [ ] Review TCP/TLS connection establishment time
- [ ] Consider IP addressing instead of DNS for critical paths

**Memory Leaks / Growing Transaction Counts:**
- [ ] Verify Timer D/K firing correctly
- [ ] Check for timer processing thread issues
- [ ] Review sofia status output for stuck transactions
- [ ] Restart profile if transactions stuck in completed state

## 9. Conclusion

FreeSWITCH's Sofia-SIP implementation provides a robust, RFC-compliant timer system with careful attention to:

- **Transport-specific behavior**: UDP retransmissions vs TCP reliability
- **Exponential backoff**: Efficient network usage with Timer A/E
- **DNS integration**: Seamless SRV failover with timer coordination
- **Configurability**: Profile-level timer tuning for diverse networks
- **Robustness**: Wraparound handling, minimum enforcement, maximum retransmit limits

Understanding these timer mechanisms is critical for:
- Diagnosing call failures and timeouts
- Optimizing for specific network conditions
- Implementing SIP-based services reliably
- Troubleshooting interoperability issues

**Key Takeaway**: Sofia-SIP timers work correctly by default. Tune only when you have measured network characteristics that justify changes from RFC 3261 defaults.

---

**Document Version**: 1.0  
**Last Updated**: 2025-10-30  
**Author**: FreeSWITCH Sofia Expert Agent  
**Based on**: FreeSWITCH source code analysis (integration branch)
