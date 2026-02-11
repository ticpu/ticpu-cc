# FreeSWITCH Variables Complete Guide

## Introduction

Covers: expansion timing (`$` vs `$$`), scopes (set/export/bridge_export/set_global), bridge string syntax (`<>`, `{}`, `[]`), action timing (inline), transfer mechanisms.

## Variable Expansion Timing

FreeSWITCH has two types of variable expansion that occur at different times.

### `$` Variables (Single Dollar) - Runtime Evaluation

**When Evaluated:** During call execution (runtime)

**How Set:**
- Dialplan applications: `set`, `export`, `set_profile_var`
- Channel-specific variables
- Dynamic call state

**Characteristics:**
- Evaluated when the call is active
- Each call can have different values
- Can change during call execution
- Most dialplan variables use this

**Examples:**
```xml
<action application="set" data="caller=${caller_id_number}"/>
<action application="log" data="INFO Caller is ${caller_id_number}"/>
```

**Common Runtime Variables:**
- `${caller_id_number}`
- `${destination_number}`
- `${uuid}`
- `${sip_from_user}`
- `${hangup_cause}`

### `$$` Variables (Double Dollar) - Parse-Time Evaluation

**When Evaluated:** When XML is loaded/compiled (reloadxml, startup)

**How Set:**
- XML preprocessing: `<X-PRE-PROCESS cmd="set" data="var=value"/>`
- FreeSWITCH API: `global_setvar var value`
- Configuration files (vars.xml)

**Characteristics:**
- Evaluated once when configuration loads
- Same value for all calls
- System-wide settings
- Better performance (no per-call expansion)

**Examples:**
```xml
<!-- In vars.xml or dialplan -->
<X-PRE-PROCESS cmd="set" data="domain=example.com"/>
<X-PRE-PROCESS cmd="set" data="local_ip=$${local_ip_v4}"/>

<!-- Used in dialplan -->
<action application="set" data="my_domain=$${domain}"/>
<!-- After XML load, this becomes: -->
<action application="set" data="my_domain=example.com"/>
```

**Common Parse-Time Variables:**
- `$${local_ip_v4}`
- `$${base_dir}`
- `$${sounds_dir}`
- `$${hold_music}`

### When to Use Each Type

**Use `$` (runtime) for:**
- Call-specific data (caller ID, destination, timestamps)
- Values that change during call execution
- Variables set by dialplan logic
- Any dynamic call state
- Data from current channel

**Use `$$` (parse-time) for:**
- Configuration values that don't change per call
- System settings (domain, IP addresses, paths)
- Values shared across all calls
- Performance optimization (expand once at load, not per call)
- Static system configuration

### Example: Mixing Both Types

```xml
<!-- Parse-time: Set system domain (in vars.xml) -->
<X-PRE-PROCESS cmd="set" data="domain=pbx.example.com"/>
<X-PRE-PROCESS cmd="set" data="recordings_dir=/var/spool/recordings"/>

<!-- Runtime: Use system values with call-specific data -->
<extension name="record_call">
  <condition field="destination_number" expression="^(\d{4})$">
    <!-- This uses parse-time domain and runtime variables -->
    <action application="set" data="record_file=$${recordings_dir}/${strftime(%Y-%m-%d)}_${uuid}.wav"/>
    <action application="bridge" data="user/$1@$${domain}"/>
  </condition>
</extension>
```

**At XML load time, becomes:**
```xml
<action application="set" data="record_file=/var/spool/recordings/${strftime(%Y-%m-%d)}_${uuid}.wav"/>
<action application="bridge" data="user/$1@pbx.example.com"/>
```

**At runtime, becomes:**
```xml
<action application="set" data="record_file=/var/spool/recordings/2025-01-15_abc123-def456.wav"/>
<action application="bridge" data="user/1234@pbx.example.com"/>
```

---

## Variable Scopes

FreeSWITCH has multiple variable scopes that control where and how variables are accessible.

### Scope Levels

#### 1. Local (Channel) Scope

Variables that exist only on one call leg.

**Set By:** `set` application

**Syntax:**
```xml
<action application="set" data="my_var=value"/>
```

**Characteristics:**
- Exists only on current channel
- NOT automatically transferred to B-leg
- Available immediately in dialplan
- Can be used in conditions and other applications

**Use Cases:**
- Temporary calculations
- Internal state tracking
- Loop counters
- Flags for logic flow

#### 2. Export Scope (A-leg → B-leg)

Variables transferred from A-leg to B-leg during call creation.

**Set By:** `export` application

**Syntax:**
```xml
<action application="export" data="my_var=value"/>
```

**Characteristics:**
- Variable is set on A-leg
- Variable name is added to `export_vars` list
- Variable will be copied to B-leg when bridge/originate creates B-leg
- One-directional: A-leg → B-leg only
- Transfer happens during B-leg creation, not during active bridge

**Use Cases:**
- SIP headers for outbound leg
- Codec preferences for B-leg
- Call tracking information
- Caller ID modifications

#### 3. Bridge Export Scope (A-leg ↔ B-leg)

Variables that transfer bidirectionally during an active bridge.

**Set By:** `bridge_export` application

**Syntax:**
```xml
<action application="bridge_export" data="my_var=value"/>
```

**Characteristics:**
- Variable is set on current channel
- Variable name is added to `bridge_export_vars` list
- When channels are bridged, the variable is copied BOTH ways
- Bidirectional: A-leg ↔ B-leg
- Transfer happens when `switch_ivr_signal_bridge()` is called

**Use Cases:**
- Variables that need to be synchronized between legs
- Call tracking information that should be consistent
- Shared state between bridged calls
- Recording paths accessible by both legs

#### 4. Global Scope (System-Wide)

Variables accessible by all channels system-wide.

**Set By:** `set_global` application

**Syntax:**
```xml
<action application="set_global" data="global_var=value"/>
```

**Characteristics:**
- Available to ALL channels on the system
- Persists until FreeSWITCH restart or explicit deletion
- Not channel-specific
- Can cause race conditions if misused

**Use Cases:**
- Emergency mode flags
- System-wide maintenance mode
- Temporary feature toggles
- Shared counters (use with caution)

### How Variables Are Tracked

FreeSWITCH maintains special tracking variables:

- **`export_vars`**: Comma-separated list of variables to export to B-leg during origination
- **`bridge_export_vars`**: Comma-separated list of variables to export bidirectionally during bridge

When you use `export` or `bridge_export` applications, FreeSWITCH:
1. Sets the variable on the current channel
2. Adds the variable name to the appropriate tracking list (`export_vars` or `bridge_export_vars`)

---

## Setting Variables

### Basic Variable Setting

#### Using `set` Application

```xml
<action application="set" data="variable_name=value"/>
```

**Standard Evaluation:**
- Variable is set during execution phase
- Expansions are evaluated when action executes
- Not available for condition checking in same extension

**Inline Evaluation:**
```xml
<action application="set" data="variable_name=value" inline="true"/>
```
- Variable is set during planning phase
- Available immediately for subsequent condition checking
- Has performance overhead
- **Use sparingly** - see [Action Execution Timing](#action-execution-timing)

#### Using `export` Application

```xml
<action application="export" data="variable_name=value"/>
```

Sets variable on A-leg AND marks it for export to B-leg.

#### Using `bridge_export` Application

```xml
<action application="bridge_export" data="variable_name=value"/>
```

Sets variable and marks it for bidirectional export during bridge.

#### Using `set_global` Application

```xml
<action application="set_global" data="variable_name=value"/>
```

Sets a system-wide global variable.

### Variable Setting Patterns

#### Multiple Variables at Once

Use `multiset` to set several related variables efficiently:

```xml
<action application="multiset" data="var1=value1 var2=value2 var3=value3"/>
```

This is more readable than separate `set` actions when defining related state.

#### From API Results

Variables can be populated from API command output at call time:

```xml
<action application="set" data="result=${api_command arg1 arg2}"/>
```

Useful for dynamic lookups (database queries, external services) that change per call.

#### With Conditional Logic

Variables can be set only when specific conditions match:

```xml
<condition field="${caller_id_number}" expression="^1000$" break="never">
  <action application="set" data="is_admin=true"/>
</condition>
```

This allows role-based variable assignment based on caller identity or other runtime state.

---

## Checking Variables

### Variable Existence

```xml
<!-- Check if variable exists and is not empty -->
<condition field="${my_var}" expression="^.+$" break="on-false">
  <action application="log" data="INFO Variable exists: ${my_var}"/>
</condition>

<!-- Check if variable is empty or doesn't exist -->
<condition field="${my_var}" expression="^$" break="on-true">
  <action application="log" data="WARNING Variable is empty"/>
</condition>
```

### Exact Value Match

```xml
<condition field="${my_var}" expression="^specific_value$" break="on-false">
  <action application="log" data="INFO Variable matches"/>
</condition>
```

### Pattern Matching

```xml
<condition field="${caller_id_number}" expression="^(\d{10})$" break="on-false">
  <action application="set" data="extracted_number=$1"/>
</condition>
```

### Multiple Variable Conditions

```xml
<!-- AND logic: All conditions must match -->
<condition field="${var1}" expression="^value1$" break="on-false"/>
<condition field="${var2}" expression="^value2$" break="on-false">
  <action application="log" data="INFO Both conditions matched"/>
</condition>

<!-- OR logic: Any condition matches -->
<condition regex="any" break="on-false">
  <regex field="${var1}" expression="^value1$"/>
  <regex field="${var2}" expression="^value2$"/>
  <action application="log" data="INFO At least one condition matched"/>
</condition>
```

---

## Variable Prefixes

### `nolocal:` Prefix (Export Without Local)

The `nolocal:` prefix tells FreeSWITCH to export a variable to the B-leg but NOT set it on the A-leg.

**Syntax:**
```xml
<action application="export" data="nolocal:variable_name=value"/>
```

**Behavior:**
- Variable is added to `export_vars` list
- Variable is NOT set on A-leg channel
- Variable WILL be set on B-leg during origination
- Logged as "(REMOTE ONLY)" in debug output

**Use Cases:**
- Setting SIP headers that should only appear on B-leg: `nolocal:sip_h_X-Custom=value`
- Configuring B-leg codec preferences without affecting A-leg
- Setting B-leg-specific dial parameters

**Example:**
```xml
<!-- Set a SIP header only on the outbound leg -->
<action application="export" data="nolocal:sip_h_X-CallType=outbound"/>
<action application="bridge" data="sofia/external/5551234@carrier.com"/>
```

**Result:**
- A-leg: `sip_h_X-CallType` is NOT set
- B-leg: `sip_h_X-CallType=outbound` IS set

**Alternative Syntax:**
FreeSWITCH also recognizes `_nolocal_` prefix (underscore-based):
```xml
<action application="export" data="_nolocal_variable_name=value"/>
```

### `origination_*` Prefix

Special variables that control B-leg creation:

```xml
<action application="export" data="nolocal:origination_caller_id_name=John Doe"/>
<action application="export" data="nolocal:origination_caller_id_number=5551234"/>
<action application="export" data="nolocal:origination_privacy=hide_name,hide_number"/>
```

**Common `origination_*` Variables:**
- `origination_caller_id_name` → B-leg caller ID name
- `origination_caller_id_number` → B-leg caller ID number
- `origination_ani` → B-leg ANI
- `origination_privacy` → B-leg privacy settings
- `origination_timeout` → B-leg timeout

### `sip_h_*` Prefix

Variables that become SIP headers:

```xml
<action application="export" data="nolocal:sip_h_X-Custom-Header=MyValue"/>
```

**Conventions:**
- `sip_h_*` → Standard SIP headers
- `sip_h_X-*` → Custom SIP headers (X- prefix)
- `sip_ph_*` → SIP P-headers (P-Asserted-Identity, etc.)

---

## Bridge String Variable Syntax

When using `bridge` or `originate`, you can set variables inline using different bracket types.

### Three Bracket Types

#### 1. Ultra-Global Variables: `<var=value>`

Variables in `<>` brackets are "ultra-global" and apply to ALL legs in the originate string.

**Syntax:**
```xml
<action application="bridge" data="<var1=value1,var2=value2>sofia/profile/dest"/>
```

**Behavior:**
- Variables are added to the base `var_event`
- Applied to ALL endpoints in the bridge string
- Parsed FIRST, before `{}` brackets
- Multiple `<>` blocks are allowed and merged

**Example:**
```xml
<action application="bridge" data="<ignore_early_media=true>sofia/profile/dest1|sofia/profile/dest2"/>
```
Both `dest1` and `dest2` will have `ignore_early_media=true`.

#### 2. Global Variables: `{var=value}`

Variables in `{}` brackets apply to all legs in the current OR group.

**Syntax:**
```xml
<action application="bridge" data="{var1=value1,var2=value2}sofia/profile/dest"/>
```

**Behavior:**
- Variables are added to the base `var_event`
- Applied to all endpoints in the pipe-separated group
- Parsed AFTER `<>` brackets
- Multiple `{}` blocks are allowed and merged
- These become "origination" variables on the B-leg

**Example:**
```xml
<action application="bridge" data="{origination_caller_id_name=John}sofia/profile/dest1,sofia/profile/dest2"/>
```
Both `dest1` and `dest2` (forked simultaneously) will have the caller ID name "John".

**Sequential Groups:**
```xml
<action application="bridge" data="{timeout=10}sofia/profile/dest1|{timeout=20}sofia/profile/dest2"/>
```
- First group tries `dest1` with 10-second timeout
- If that fails, second group tries `dest2` with 20-second timeout

#### 3. Per-Leg Variables: `[var=value]`

Variables in `[]` brackets apply ONLY to the specific endpoint they precede.

**Syntax:**
```xml
<action application="bridge" data="[var1=value1,var2=value2]sofia/profile/dest"/>
```

**Behavior:**
- Variables apply to the immediately following endpoint ONLY
- Takes precedence over `{}` and `<>` variables
- Multiple `[]` blocks are allowed per endpoint and merged
- Most specific scope

**Example:**
```xml
<action application="bridge" data="[sip_h_X-Route=primary]sofia/profile/dest1,[sip_h_X-Route=backup]sofia/profile/dest2"/>
```
- `dest1` has SIP header `X-Route: primary`
- `dest2` has SIP header `X-Route: backup`

### Combining Bracket Types

You can combine all three types for fine-grained control:

```xml
<action application="bridge" data="<ignore_early_media=true>{origination_caller_id_name=System}[sip_h_X-Priority=high]sofia/profile/dest1,[sip_h_X-Priority=low]sofia/profile/dest2"/>
```

**Processing Order:**
1. `<ignore_early_media=true>` - Applied to ALL endpoints
2. `{origination_caller_id_name=System}` - Applied to dest1 and dest2
3. `[sip_h_X-Priority=high]` - Applied only to dest1
4. `[sip_h_X-Priority=low]` - Applied only to dest2

**Result:**
- Both endpoints: `ignore_early_media=true`, `origination_caller_id_name=System`
- `dest1`: Also has `sip_h_X-Priority=high`
- `dest2`: Also has `sip_h_X-Priority=low`

### Using `nolocal:` in Bridge Strings

You can use `nolocal:` in bridge string variables:

```xml
<action application="bridge" data="{nolocal:sip_h_X-Custom=value}sofia/external/dest"/>
```

This sets the header only on B-leg, not on A-leg.

---

## Variable Transfer Mechanisms

### During Originate (Bridge Creation)

When FreeSWITCH creates a B-leg via `bridge` or `originate`:

**1. Export Variables Processing (`export_vars`):**
- FreeSWITCH reads the `export_vars` list from A-leg
- For each variable name in the list:
  - If variable has `nolocal:` or `_nolocal_` prefix: Skip setting on A-leg
  - Get variable value from A-leg
  - Set variable on B-leg (or in origination event)
- Happens in `switch_channel_process_export()`

**2. Origination Variables:**
- Variables from `<>`, `{}`, and `[]` brackets are collected
- `[]` variables override `{}` which override `<>` for same variable name
- Special `origination_*` variables affect B-leg creation

**3. Merge Order:**
```
Base var_event (from export_vars)
↓
+ Ultra-global variables (<>)
↓
+ Global variables ({})
↓
+ Per-leg variables ([])
↓
Final originate_var_event passed to B-leg creation
```

### During Active Bridge

When two channels are bridged (already active):

**1. Bridge Export Variables Processing (`bridge_export_vars`):**
- FreeSWITCH reads `bridge_export_vars` from BOTH channels
- Variables listed are copied bidirectionally:
  - A-leg → B-leg
  - B-leg → A-leg
- Happens in `check_bridge_export()` function
- Called during `switch_ivr_signal_bridge()`

**2. Timing:**
- Occurs when bridge is established, not when B-leg is created
- Happens AFTER both legs exist and are being connected

### Critical Timing Difference

**`export` Application:**
- Variables are transferred during B-leg CREATION
- Happens when `bridge` or `originate` spawns new channel
- One-time transfer at origination

**`bridge_export` Application:**
- Variables are transferred during bridge ESTABLISHMENT
- Happens when existing channels are being connected
- Can occur multiple times if channels are re-bridged

---

## Action Execution Timing

Understanding when variables are set and when they're available is critical for dialplan logic.

### Two Execution Phases

#### Planning Phase (Condition Evaluation)

During planning phase:
- Conditions are evaluated
- Field expressions are expanded
- Action queue is built
- **`inline` actions execute here**

#### Execution Phase

During execution phase:
- Queued actions execute in order
- Regular `set` happens here
- Media operations occur here

### Critical Difference: inline vs Regular

**WITHOUT `inline`:**
```xml
<condition field="destination_number" expression="^(\d+)$">
  <action application="set" data="dest=$1"/>
  <!-- dest is queued but not set yet -->

  <condition field="${dest}" expression="^\d+$">
    <!-- dest is empty during planning! This won't work! -->
  </condition>
</condition>
```

**WITH `inline`:**
```xml
<condition field="destination_number" expression="^(\d+)$">
  <action application="set" data="dest=$1" inline="true"/>
  <!-- dest is set RIGHT NOW during planning -->

  <condition field="${dest}" expression="^\d+$">
    <!-- dest has value! Can be checked -->
    <action application="log" data="Destination: ${dest}"/>
  </condition>
</condition>
```

### When to Use `inline="true"`

**Use inline when:**
- Setting a variable that subsequent conditions need to check
- Extracting data for immediate use in routing decisions
- Nested conditions that depend on extracted values

**Do NOT use inline when:**
- Normal variable setting for execution phase
- Variables that don't affect condition evaluation
- Performance-sensitive paths (inline has overhead)

### Example: inline for Nested Logic

```xml
<condition field="destination_number" expression="^9(\d+)$" break="on-false">
  <!-- Extract number immediately -->
  <action application="set" data="outbound_number=$1" inline="true"/>

  <!-- Now we can check the extracted number -->
  <condition field="${outbound_number}" expression="^911$" break="on-true">
    <!-- Emergency! -->
    <action application="bridge" data="sofia/gateway/emergency/911"/>
  </condition>

  <!-- Not emergency, route normally -->
  <condition break="on-false">
    <action application="bridge" data="sofia/gateway/provider/${outbound_number}"/>
  </condition>
</condition>
```

### Best Practice: Avoid inline When Possible

✅ **BETTER: Use sequential extensions**
```xml
<extension name="extract_number" continue="true">
  <condition field="destination_number" expression="^9(\d+)$" break="on-false">
    <action application="set" data="outbound_number=$1"/>
  </condition>
</extension>

<extension name="route_emergency">
  <condition field="${outbound_number}" expression="^911$" break="on-false">
    <action application="bridge" data="sofia/gateway/emergency/911"/>
  </condition>
</extension>

<extension name="route_normal">
  <condition field="${outbound_number}" expression="^\d+$" break="on-false">
    <action application="bridge" data="sofia/gateway/provider/${outbound_number}"/>
  </condition>
</extension>
```

---

## Practical Examples

### Setting SIP Headers for Outbound Only

**Goal:** Add custom SIP header to outbound leg only, not A-leg.

```xml
<extension name="outbound_with_header">
  <condition field="destination_number" expression="^(\d{10})$" break="on-false">
    <!-- Set SIP header only on outbound leg -->
    <action application="export" data="nolocal:sip_h_X-Customer-ID=12345"/>
    <action application="export" data="nolocal:sip_h_X-Call-Type=outbound"/>

    <action application="bridge" data="sofia/external/$1@carrier.com"/>
  </condition>
</extension>
```

### Different Codec Preferences Per Leg

**Goal:** Prefer different codecs on A-leg vs B-leg.

```xml
<extension name="asymmetric_codecs">
  <condition field="destination_number" expression="^8000$" break="on-false">
    <!-- A-leg prefers PCMU -->
    <action application="set" data="absolute_codec_string=PCMU,PCMA"/>

    <!-- B-leg should prefer opus -->
    <action application="export" data="nolocal:absolute_codec_string=opus,PCMU,PCMA"/>

    <action application="bridge" data="sofia/internal/8001"/>
  </condition>
</extension>
```

### Per-Gateway Settings in Sequential Failover

**Goal:** Try multiple gateways with different settings.

```xml
<extension name="multi_gateway_failover">
  <condition field="destination_number" expression="^91(\d{10})$" break="on-false">
    <action application="set" data="continue_on_fail=true"/>
    <action application="set" data="hangup_after_bridge=false"/>

    <!-- Try primary gateway with short timeout -->
    <action application="bridge" data="{origination_timeout=10}[sip_h_X-Route=primary]sofia/gateway/primary/$1"/>

    <!-- Try secondary gateway with longer timeout -->
    <action application="bridge" data="{origination_timeout=20}[sip_h_X-Route=secondary]sofia/gateway/secondary/$1"/>

    <!-- Try tertiary gateway with longest timeout -->
    <action application="bridge" data="{origination_timeout=30}[sip_h_X-Route=tertiary]sofia/gateway/tertiary/$1"/>
  </condition>
</extension>
```

### Parallel Forking with Per-Endpoint Variables

**Goal:** Ring multiple endpoints simultaneously with different settings.

```xml
<extension name="find_me_follow_me">
  <condition field="destination_number" expression="^7000$" break="on-false">
    <!-- Global settings for all endpoints -->
    <action application="bridge" data="{call_timeout=25,ignore_early_media=true}[leg_timeout=15]user/1000@domain,[leg_timeout=20]user/1001@domain,[leg_timeout=25]user/1002@domain"/>
  </condition>
</extension>
```

### Bidirectional Variable Sharing

**Goal:** Share call recording path between both legs.

```xml
<extension name="shared_recording">
  <condition field="destination_number" expression="^6000$" break="on-false">
    <!-- Set recording path and export bidirectionally -->
    <action application="set" data="recording_path=/var/recordings/${uuid}.wav"/>
    <action application="bridge_export" data="recording_path"/>

    <action application="bridge" data="user/6001@domain"/>

    <!-- After bridge, both legs can access recording_path -->
  </condition>
</extension>
```

### Complex Multi-Scope Variables

**Goal:** Set variables at different scopes for complex routing.

```xml
<extension name="complex_routing">
  <condition field="destination_number" expression="^5(\d{3})$" break="on-false">
    <!-- Parse-time variable (system-wide) -->
    <!-- Assume this is set in vars.xml: -->
    <!-- $${domain} = example.com -->

    <!-- System-wide setting (affects all calls) -->
    <action application="set_global" data="emergency_mode=false"/>

    <!-- A-leg only -->
    <action application="set" data="call_direction=inbound"/>

    <!-- Export to B-leg -->
    <action application="export" data="original_destination=$1"/>
    <action application="export" data="nolocal:sip_h_X-Original-Dest=$1"/>

    <!-- Bidirectional -->
    <action application="bridge_export" data="call_uuid=${uuid}"/>

    <!-- Bridge with ultra-global, global, and per-leg variables -->
    <action application="bridge" data="<ignore_early_media=true>{origination_caller_id_name=PBX}[sip_h_X-Priority=high]sofia/internal/$1@$${domain},[sip_h_X-Priority=normal]sofia/internal/$1@$${domain}"/>
  </condition>
</extension>
```

**Processing:**
1. Parse-time: `$${domain}` → `example.com`
2. Global: `emergency_mode=false` (all channels)
3. A-leg: `call_direction=inbound` (local only)
4. A-leg → B-leg: `original_destination=$1` (export)
5. B-leg only: `sip_h_X-Original-Dest=$1` (nolocal export)
6. A-leg ↔ B-leg: `call_uuid=${uuid}` (bridge_export)
7. All B-legs: `ignore_early_media=true` (ultra-global)
8. All B-legs: `origination_caller_id_name=PBX` (global)
9. First server: `sip_h_X-Priority=high` (per-leg)
10. Second server: `sip_h_X-Priority=normal` (per-leg)

---

## Best Practices

### Prefer `set` + `export nolocal:` Over Plain `export`

**Recommended Pattern:**
```xml
<action application="set" data="my_var=value"/>
<action application="export" data="nolocal:sip_h_X-Custom=value"/>
```

**Why:**
- `set` keeps A-leg variable local (cleaner A-leg state)
- `export nolocal:` sends only what B-leg needs
- Avoids polluting A-leg with B-leg-specific variables
- Makes intent explicit and debugging easier

**Avoid:**
```xml
<action application="export" data="sip_h_X-Custom=value"/>  <!-- Sets on BOTH legs -->
```

### Minimize Custom Variables

**Prefer standard FreeSWITCH variables over custom ones:**
- Use built-in variables when possible (see `freeswitch_channel_variables.md`)
- Avoid creating custom `sip_h_*` headers unless necessary
- Standard variables are better documented and supported
- Custom variables add cognitive load for future maintainers

**Example - Use standard variables:**
```xml
<!-- Good: Use standard origination_caller_id_* -->
<action application="export" data="nolocal:origination_caller_id_name=Company"/>

<!-- Avoid: Custom header when standard exists -->
<action application="export" data="nolocal:sip_h_X-Caller-Name=Company"/>
```

---

## Common Patterns

### Pattern 1: Clean B-leg SIP Headers

```xml
<!-- Remove unwanted headers, add custom ones for B-leg only -->
<action application="set" data="sip_copy_custom_headers=false"/>
<action application="export" data="nolocal:sip_h_X-My-Header=value"/>
<action application="bridge" data="sofia/external/dest"/>
```

**Why:**
- `sip_copy_custom_headers=false` prevents automatic header copying
- `nolocal:sip_h_X-*` adds only desired headers to B-leg
- Results in clean, controlled SIP headers on outbound

### Pattern 2: Call Tracking with Bidirectional Export

```xml
<!-- Track call across transfers -->
<action application="set" data="initial_uuid=${uuid}"/>
<action application="bridge_export" data="initial_uuid"/>
<action application="bridge_export" data="account_code=${account_code}"/>
<action application="bridge" data="user/dest"/>
```

**Why:**
- `bridge_export` ensures both legs have tracking variables
- If either leg is transferred, variables persist
- Useful for billing and CDR correlation

### Pattern 3: Gateway-Specific Retry Logic

```xml
<action application="set" data="continue_on_fail=NORMAL_TEMPORARY_FAILURE,TIMEOUT"/>
<action application="bridge" data="{origination_timeout=10}sofia/gateway/gw1/dest|{origination_timeout=15}sofia/gateway/gw2/dest"/>
```

**Why:**
- Different timeouts per gateway
- Controlled failure handling
- Sequential failover with appropriate timing

### Pattern 4: Recording Path Export

```xml
<!-- Set up recording that both legs can reference -->
<action application="set" data="RECORD_STEREO=true"/>
<action application="set" data="record_path=/var/spool/freeswitch/recordings/${strftime(%Y-%m-%d-%H-%M-%S)}_${uuid}.wav"/>
<action application="export" data="record_path"/>
<action application="record_session" data="${record_path}"/>
<action application="bridge" data="user/dest"/>
```

**Why:**
- Both legs can access `record_path` for logging/CDR
- B-leg can use path for post-call processing
- Consistent recording reference across legs

### Pattern 5: Privacy Control

```xml
<!-- Hide caller ID on outbound, but preserve internally -->
<action application="export" data="nolocal:origination_privacy=hide_name,hide_number"/>
<action application="export" data="nolocal:sip_h_Privacy=id"/>
<action application="bridge" data="sofia/external/dest@carrier"/>
```

**Why:**
- A-leg retains full caller ID information
- B-leg (carrier) receives privacy headers
- Complies with caller ID blocking requirements

### Pattern 6: Parse-Time Optimization

```xml
<!-- In vars.xml -->
<X-PRE-PROCESS cmd="set" data="recordings_base=/var/spool/freeswitch/recordings"/>
<X-PRE-PROCESS cmd="set" data="default_gateway=sofia/gateway/primary"/>

<!-- In dialplan -->
<action application="set" data="record_file=$${recordings_base}/${uuid}.wav"/>
<action application="bridge" data="$${default_gateway}/${destination_number}"/>
```

**Why:**
- Parse-time variables expanded once at load
- Better performance (no per-call expansion)
- Easy to change system-wide settings
- Centralized configuration

---

## Troubleshooting

### Variable Not Transferred to B-leg

**Symptom:** Variable set on A-leg doesn't appear on B-leg.

**Diagnosis:**
```xml
<!-- Check if variable is being exported -->
<action application="info"/>  <!-- Shows all variables including export_vars -->
```

**Solutions:**

❌ WRONG:
```xml
<action application="set" data="my_var=value"/>
<action application="bridge" data="user/dest"/>
```

✅ CORRECT:
```xml
<action application="export" data="my_var=value"/>
<action application="bridge" data="user/dest"/>
```

### Variable Set Too Late

**Symptom:** Using `export` after `bridge` doesn't work.

❌ WRONG:
```xml
<action application="bridge" data="user/dest"/>
<action application="export" data="my_var=value"/>
```

✅ CORRECT:
```xml
<action application="export" data="my_var=value"/>
<action application="bridge" data="user/dest"/>
```

### Variable Set on Both Legs Unintentionally

**Symptom:** Variable appears on A-leg but should only be on B-leg.

❌ WRONG:
```xml
<action application="export" data="sip_h_X-Custom=value"/>
```

✅ CORRECT:
```xml
<action application="export" data="nolocal:sip_h_X-Custom=value"/>
```

### Per-Leg Variables Affecting All Legs

**Symptom:** `[]` variable appearing on all forked legs.

❌ WRONG:
```xml
<action application="bridge" data="[priority=high]user/1000,user/1001,user/1002"/>
```

✅ CORRECT:
```xml
<action application="bridge" data="[priority=high]user/1000,[priority=normal]user/1001,[priority=low]user/1002"/>
```

### Parse-Time Variable Not Expanding

**Symptom:** `$${var}` appears literally in logs instead of expanding.

❌ WRONG:
```xml
<!-- Trying to create parse-time variable in dialplan -->
<action application="set_global" data="my_parse_var=value"/>
<action application="log" data="Value: $${my_parse_var}"/>
```

✅ CORRECT:
```xml
<!-- In vars.xml or using X-PRE-PROCESS -->
<X-PRE-PROCESS cmd="set" data="my_parse_var=value"/>
<action application="log" data="Value: $${my_parse_var}"/>
```

Or use runtime global access:
```xml
<action application="set_global" data="my_var=value"/>
<action application="log" data="Value: ${global(my_var)}"/>
```

### Debugging Variables

**View all channel variables:**
```xml
<action application="info"/>
```
Shows all variables on current channel, including:
- `export_vars` - List of variables marked for export
- `bridge_export_vars` - List of variables marked for bridge export
- All channel variables and values

**Check specific variable:**
```xml
<action application="log" data="INFO Variable value: ${my_var}"/>
```

**Trace variable export:**
Enable DEBUG logging in FreeSWITCH console:
```
fs_cli> console loglevel debug
```
Look for log lines:
```
EXPORT (export_vars) variable_name=[value]
EXPORT (export_vars) (REMOTE ONLY) variable_name=[value]
```

---

## Regex Capture Groups in Dialplans

### Overview

When FreeSWITCH XML dialplan conditions use regular expressions with capture groups `(...)`, the matched values are accessible as `$1`, `$2`, `$3`, etc.

**Key Facts:**
- Capture groups are stored in a channel variable array called `DP_MATCH`
- `$0` = full matched string, `$1` = first capture group, `$2` = second, etc.
- These are channel-scoped variables
- Source: `src/mod/dialplans/mod_dialplan_xml/mod_dialplan_xml.c` lines 507-509

### Capture Group Behavior

#### 1. Sequential Conditions: Capture groups are OVERWRITTEN

When a condition with capture groups matches, it clears and replaces all previous captures.

**Pattern:**
```xml
<extension name="sequential_conditions">
  <condition field="${caller_id_number}" expression="^(555)(\d{4})$">
    <!-- Sets: $1=555, $2=1234 -->
    <action application="log" data="INFO Area: $1, Number: $2"/>
  </condition>

  <condition field="${destination_number}" expression="^(9)(\d{10})$">
    <!-- If this matches: $1=9, $2=(first 10 digits) -->
    <!-- Previous values (555, 1234) are LOST -->
    <action application="log" data="INFO Prefix: $1, Number: $2"/>
  </condition>
</extension>
```

**Code:** `mod_dialplan_xml.c:508` clears `DP_MATCH` before setting new captures when a condition with `(...)` matches.

#### 2. Nested Conditions: Child REPLACES parent captures

**Pattern:**
```xml
<extension name="nested_conditions">
  <condition field="${caller_id_number}" expression="^(555)(\d{4})$">
    <!-- Parent: $1=555, $2=1234 -->

    <condition field="${destination_number}" expression="^(1)(\d{3})$">
      <!-- Nested match: $1 becomes "1", $2 becomes 3-digit extension -->
      <!-- Parent's captures are LOST -->
      <action application="log" data="INFO Extension: $1$2"/>
    </condition>
  </condition>
</extension>
```

**Code:** Nested conditions recursively call `parse_exten()` with same channel (`mod_dialplan_xml.c:583`), so they share the same `DP_MATCH` variable.

#### 3. Persistence: Captures ONLY reset when new match has captures

Capture groups persist until overwritten by another matching condition that contains `(...)`.

**Pattern:**
```xml
<extension name="capture_persistence">
  <condition field="${caller_id_number}" expression="^(555)(\d{4})$">
    <!-- Sets: $1=555, $2=1234 -->
  </condition>

  <condition field="${some_var}" expression="^yes$">
    <!-- No capture groups - $1 and $2 still available -->
    <action application="log" data="INFO Still: $1, $2"/>
  </condition>
</extension>
```

**Code:** Reset only occurs when `expression` contains `(` (`mod_dialplan_xml.c:507`).

#### 4. No Stacking: Single shared set, each match replaces previous

To preserve captures across multiple regex matches, save to named variables.

**Pattern:**
```xml
<extension name="preserve_captures">
  <condition field="${caller_id_number}" expression="^(555)(\d{4})$">
    <!-- Save immediately with inline -->
    <action application="set" data="caller_area=$1" inline="true"/>
    <action application="set" data="caller_num=$2" inline="true"/>

    <condition field="${destination_number}" expression="^(1)(\d{3})$">
      <!-- $1 and $2 now contain new values -->
      <!-- But caller_area and caller_num preserved -->
      <action application="bridge" data="user/$1$2@domain"/>
    </condition>
  </condition>
</extension>
```

**Code:** All conditions share one `DP_MATCH` on the channel (`mod_dialplan_xml.c:508-509`).

#### 5. Scope and Lifecycle

**Scope:** Channel-level variables
- Accessible across conditions and extensions (with `continue="true"`)
- NOT transferred to B-leg automatically (need `export`)

**Lifecycle:** Created on match → Exists until next match with captures → Destroyed on hangup

**Multiple `<regex>` Elements:**
When using `regex="all"/"any"/"xor"` with multiple `<regex>` children:
```c
// src/mod/dialplans/mod_dialplan_xml/mod_dialplan_xml.c, line 234
switch_channel_del_variable_prefix(channel, "DP_REGEX_MATCH");
```
Each `<regex>` creates numbered variables: `${DP_REGEX_MATCH_1_1}`, `${DP_REGEX_MATCH_2_1}`, etc.

### Usage Patterns

#### Basic Capture

```xml
<extension name="outbound">
  <condition field="destination_number" expression="^9(\d{10})$">
    <action application="bridge" data="sofia/gateway/provider/$1"/>
  </condition>
</extension>
```

**Code:** `mod_dialplan_xml.c:537` calls `switch_perform_substitution()` to replace `$1` in action data.

#### Multiple Captures

```xml
<extension name="format_number">
  <condition field="destination_number" expression="^(\d{3})(\d{3})(\d{4})$">
    <!-- $1=555, $2=123, $3=4567 -->
    <action application="set" data="formatted=$1-$2-$3"/>
    <action application="bridge" data="sofia/gateway/provider/$1$2$3"/>
  </condition>
</extension>
```

#### Preserving Captures Across Nested Conditions

```xml
<extension name="preserve">
  <condition field="${caller_id_number}" expression="^1(555)(\d{7})$">
    <!-- Save immediately with inline to prevent loss -->
    <action application="set" data="caller_area=$1" inline="true"/>
    <action application="set" data="caller_num=$2" inline="true"/>

    <condition field="${destination_number}" expression="^(\d{4})$">
      <!-- $1 now = extension, but saved vars still available -->
      <action application="bridge" data="user/$1@domain"/>
    </condition>
  </condition>
</extension>
```

**Why inline:** Without `inline`, `set` executes after planning phase when `$1` is already overwritten by nested condition.

#### Multiple Regex Elements

```xml
<extension name="multi_regex">
  <condition regex="all">
    <regex field="${caller_id_number}" expression="^(555)(\d{4})$"/>
    <regex field="${destination_number}" expression="^(1)(\d{3})$"/>
    <action application="log" data="INFO From: ${DP_REGEX_MATCH_1_1}, To: ${DP_REGEX_MATCH_2_1}${DP_REGEX_MATCH_2_2}"/>
  </condition>
</extension>
```

**Code:** `mod_dialplan_xml.c:234` clears all `DP_REGEX_MATCH` vars, lines 342-347 create numbered variables per `<regex>` element.

### Useful Patterns

```xml
<!-- Area code extraction -->
<extension name="area_code">
  <condition field="destination_number" expression="^1?(\d{3})(\d{7})$">
    <action application="bridge" data="sofia/gateway/provider/$1$2"/>
  </condition>
</extension>

<!-- Optional prefix (9 to dial out) -->
<extension name="optional_9">
  <condition field="destination_number" expression="^(?:9|)(\d{4})$">
    <action application="bridge" data="user/$1@domain"/>
  </condition>
</extension>

<!-- International with routing by country -->
<extension name="international">
  <condition field="destination_number" expression="^011(\d{1,3})(\d+)$">
    <action application="set" data="country=$1" inline="true"/>
    <action application="set" data="number=$2" inline="true"/>

    <condition field="${country}" expression="^(1|44|61)$">
      <action application="bridge" data="sofia/gateway/intl1/${country}${number}"/>
    </condition>
  </condition>
</extension>
```

---

## Reference Tables

### Variable Expansion Type Reference

| Type | Syntax | When Evaluated | Set By | Use For |
|------|--------|----------------|--------|---------|
| Runtime | `${var}` | During call execution | Dialplan applications | Call-specific data, dynamic state |
| Parse-time | `$${var}` | XML load/compile time | X-PRE-PROCESS, global_setvar | System config, static values |

### Variable Scope Reference

| Scope | Application | Dialplan Syntax | Bridge Syntax | Direction | Timing |
|-------|-------------|-----------------|---------------|-----------|--------|
| Local | `set` | `<action application="set" data="var=val"/>` | N/A | Current channel only | Immediate |
| Export to B-leg | `export` | `<action application="export" data="var=val"/>` | `{var=val}` or `[var=val]` | A-leg → B-leg | B-leg creation |
| Export to B-leg only | `export` with `nolocal:` | `<action application="export" data="nolocal:var=val"/>` | `{nolocal:var=val}` or `[nolocal:var=val]` | B-leg only | B-leg creation |
| Bidirectional | `bridge_export` | `<action application="bridge_export" data="var=val"/>` | N/A | A-leg ↔ B-leg | Bridge establishment |
| Global (system) | `set_global` | `<action application="set_global" data="var=val"/>` | N/A | All channels | Immediate, persistent |
| Ultra-global (originate) | N/A | N/A | `<var=val>` | All B-legs in originate string | B-leg creation |

### Variable Prefix Reference

| Prefix | Usage | Effect |
|--------|-------|--------|
| `nolocal:` | `export` data | Variable exported to B-leg but NOT set on A-leg |
| `_nolocal_` | `export` data | Alternative syntax for `nolocal:` |
| `origination_*` | Bridge/originate variables | Special variables that control B-leg creation (caller ID, ANI, privacy, etc.) |
| `sip_h_*` | Variable name | SIP header variable (automatically added as header to SIP messages) |
| `sip_h_X-*` | Variable name | Custom SIP header (X- prefix) |

### Bracket Type Reference

| Bracket | Scope | Example | Use Case |
|---------|-------|---------|----------|
| `<>` | Ultra-global (all B-legs) | `<ignore_early_media=true>` | Settings that apply to entire originate |
| `{}` | Global (current OR group) | `{origination_timeout=20}` | Settings for all endpoints in parallel or sequential group |
| `[]` | Per-leg (specific endpoint) | `[sip_h_X-Priority=high]` | Settings for one specific endpoint only |

### Export Mechanism Reference

| Variable List | Set By | Used By | Direction |
|--------------|--------|---------|-----------|
| `export_vars` | `export` application | `switch_channel_process_export()` during originate | A-leg → B-leg |
| `bridge_export_vars` | `bridge_export` application | `check_bridge_export()` during bridge | A-leg ↔ B-leg |

### Action Timing Reference

| Phase | When | What Happens | inline Effect |
|-------|------|--------------|---------------|
| Planning | Condition evaluation | Conditions checked, actions queued | inline actions execute here |
| Execution | After planning | Queued actions run in order | Regular actions execute here |

---

