### Definition

A **context** is a named collection of extensions that defines a routing domain in FreeSWITCH. Think of contexts as **routing tables** - they determine what happens when a specific destination number is dialed.

### Structure

```xml
<context name="context_name">
    <extension name="extension1">
        <!-- conditions and actions -->
    </extension>
    <extension name="extension2">
        <!-- conditions and actions -->
    </extension>
</context>
```

### Key Characteristics

1. **Contexts are routing boundaries** - A call enters a context and processes extensions until:
   - A transfer moves it to another context
   - The call is bridged/answered
   - The call hangs up
   - All extensions are exhausted

2. **Contexts are NOT variable scopes** - Channel variables persist across context transfers. There is no "context variable" scope.

3. **Contexts provide security isolation** - Different contexts can have different access to resources, providing defense-in-depth security.

4. **Contexts are selected by:**
   - SIP profile configuration (inbound calls)
   - User directory (registered users)
   - Transfer application
   - Execute_extension application (stays in same context)

---

## Context as Security Boundaries

### Why Context Isolation Matters

**Principle:** Untrusted sources should never have direct access to privileged destinations.

### Security Model

```
INTERNET (Untrusted)
    ↓
Public Context (Validate + Sanitize)
    ↓
Authentication Context (Verify Identity)
    ↓
Internal Context (Trusted Operations)
    ↓
Emergency Context (911/Critical Services)
```

### Real-World Security Scenario

**Bad Design:**
```xml
<context name="public">
    <!-- DANGER: External callers can reach internal extensions -->
    <extension name="all_numbers">
        <condition field="destination_number" expression="^(\d+)$">
            <action application="bridge" data="user/$1@${domain}"/>
        </condition>
    </extension>
</context>
```

**Good Design:**
```xml
<context name="public">
    <!-- Only allow access to published numbers -->
    <extension name="main_line">
        <condition field="destination_number" expression="^8005551234$">
            <action application="transfer" data="main_menu XML ivr"/>
        </condition>
    </extension>

    <extension name="reject_others">
        <condition field="destination_number" expression=".*">
            <action application="log" data="WARNING Unauthorized access attempt: ${destination_number}"/>
            <action application="hangup" data="CALL_REJECTED"/>
        </condition>
    </extension>
</context>

<context name="ivr">
    <!-- Controlled IVR options -->
    <extension name="ivr_menu">
        <condition field="destination_number" expression="^main_menu$">
            <action application="play_and_get_digits" data="1 1 3 5000 # menu.wav invalid.wav choice ^[1-3]$"/>
            <action application="transfer" data="option_${choice} XML internal_routing"/>
        </condition>
    </extension>
</context>

<context name="internal_routing">
    <!-- Internal extensions only reachable through controlled paths -->
    <extension name="option_1">
        <condition field="destination_number" expression="^option_1$">
            <action application="bridge" data="group/sales@${domain}"/>
        </condition>
    </extension>
</context>
```

### Context Trust Levels

**Recommended hierarchy:**

| Context Type | Trust Level | Can Access | Entry Point |
|-------------|-------------|------------|-------------|
| `public` | Untrusted | Only published services | Internet/PSTN |
| `authenticated` | Verified | User-specific features | After authentication |
| `internal` | Trusted | Internal extensions | Registered users |
| `emergency` | Critical | Emergency services | Transfer only |
| `admin` | Privileged | System functions | Secure interfaces only |

---

## Transfer Mechanisms

### Understanding Transfer vs Execute_Extension

From `switch_ivr.h`:
```c
switch_ivr_session_transfer(session, extension, dialplan, context);
```

### 1. Transfer Application

**Purpose:** Move to a different context and/or extension

**Syntax:**
```xml
<action application="transfer" data="extension [dialplan [context]]"/>
```

**Behavior:**
- Changes the channel's destination
- Can change context
- Processes the target extension from the beginning
- Previous extension processing stops
- All channel variables persist

**Example:**
```xml
<!-- From public context -->
<action application="transfer" data="1000 XML internal"/>
<!-- Now processing extension "1000" in context "internal" -->
```

### 2. Execute_Extension Application

**Purpose:** Call an extension in the same or different context like a subroutine

**Syntax:**
```xml
<action application="execute_extension" data="extension XML context"/>
```

**Behavior:**
- If context specified: processes that context's extension
- If context omitted: processes extension in current context
- Returns to calling point after execution
- Variables persist
- Can be used for "subroutine" patterns

**Example:**
```xml
<extension name="main">
    <condition field="destination_number" expression="^1000$">
        <action application="set" data="return_point=main"/>
        <action application="execute_extension" data="validate_user XML utilities"/>
        <!-- Execution returns here -->
        <action application="bridge" data="user/1000@${domain}"/>
    </condition>
</extension>
```

### When to Use Each

| Scenario | Use |
|----------|-----|
| Change routing stage | `transfer` |
| Security boundary crossing | `transfer` |
| Subroutine/helper function | `execute_extension` |
| IVR menu progression | `transfer` |
| Validation check | `execute_extension` |
| Multi-stage call flow | `transfer` |

---

## Variable Behavior Across Contexts

### Critical Understanding: Variables Are Channel-Scoped

**Key Principle:** Channel variables belong to the **channel/session**, not the context.

### What Happens During Transfer

```xml
<context name="context_a">
    <extension name="set_vars">
        <condition field="destination_number" expression="^test$">
            <action application="set" data="my_var=value_from_a"/>
            <action application="set" data="context_a_data=important"/>
            <action application="transfer" data="process XML context_b"/>
        </condition>
    </extension>
</context>

<context name="context_b">
    <extension name="process">
        <condition field="destination_number" expression="^process$">
            <!-- my_var and context_a_data are STILL AVAILABLE -->
            <action application="log" data="INFO my_var=${my_var}"/>
            <action application="log" data="INFO context_a_data=${context_a_data}"/>
        </condition>
    </extension>
</context>
```

**Result:** All variables transfer with the channel. There is no "context variable" isolation.

### Why "Minimum Context Variables" Matters

The principle of **minimum context variables** means:

1. **Don't assume variables exist** - Each context should validate required variables
2. **Don't create hidden dependencies** - Context B shouldn't silently depend on Context A setting variables
3. **Make contexts self-contained** - Each context should work independently
4. **Use explicit validation** - Check for required variables at context entry

### Best Practice Pattern

```xml
<context name="target_context">
    <extension name="validate_entry">
        <!-- Validate required variables exist -->
        <condition field="${required_var}" expression="^$" break="on-true">
            <action application="log" data="ERR Context entry failed: missing required_var"/>
            <action application="hangup" data="NORMAL_UNSPECIFIED"/>
        </condition>

        <!-- Only proceed if validation passed -->
        <condition field="destination_number" expression="^(.+)$">
            <action application="log" data="INFO Processing ${destination_number} with ${required_var}"/>
            <!-- Continue processing -->
        </condition>
    </extension>
</context>
```

### Variable Export Across Bridges

**Important distinction:** Variables crossing **context boundaries** vs crossing **call leg boundaries**

```xml
<!-- Context transfer: Variables stay on same channel -->
<action application="transfer" data="ext XML other_context"/>

<!-- Bridge: Variables can be exported to B-leg -->
<action application="export" data="nolocal:my_var=value"/>
<action application="bridge" data="user/1000@${domain}"/>
```

---

## When to Create New Contexts

### Decision Criteria

Create a new context when you need:

1. **Security Isolation**
   - External vs internal calls
   - Authenticated vs unauthenticated
   - User vs admin operations

2. **Functional Separation**
   - IVR systems
   - Queue handling
   - Emergency routing
   - Conferencing
   - Voicemail

3. **Multi-Stage Workflows**
   - Registration → Authentication → Services
   - Call validation → Routing → Connection
   - Menu navigation → Option selection → Action

4. **Tenant Isolation**
   - Multi-tenant environments
   - Department separation
   - Customer segregation

5. **Different Entry Points**
   - Internet-facing SIP profile
   - Internal SIP profile
   - Gateway connections
   - API-initiated calls

### When NOT to Create New Contexts

Avoid creating contexts for:

1. **Simple routing variations** - Use extensions with conditions
2. **Time-based changes** - Use time conditions in same context
3. **User preferences** - Use variables and conditions
4. **Minor workflow branches** - Use nested conditions or execute_extension

### Agent Checklist: Should I Suggest a New Context?

Ask yourself:
- Does this cross a trust boundary? (external → internal)
- Are we mixing incompatible operations? (emergency + admin calls)
- Mixing different networks (esinet vs pstn)
- Would combining these create security risks?
- Are there different access control requirements?
- Is this a distinct stage in a multi-step process?
- Would separate contexts improve maintainability?

**If YES to 2+ questions:** Suggest separate contexts
**If NO to all:** Consider same context with extensions/conditions

---

## Context Architecture Patterns

### Pattern 1: Security-Layered Architecture

```
public (untrusted)
  ├─→ ivr (semi-trusted, limited functions)
  └─→ authenticated (verified identity)
       ├─→ internal (trusted operations)
       ├─→ emergency (critical services)
       └─→ admin (privileged functions)
```

**Implementation:**
```xml
<context name="public">
    <extension name="dids">
        <condition field="destination_number" expression="^8005551234$">
            <action application="transfer" data="main_menu XML ivr"/>
        </condition>
    </extension>
</context>

<context name="ivr">
    <extension name="main_menu">
        <condition field="destination_number" expression="^main_menu$">
            <action application="play_and_get_digits" data="1 1 3 5000 # menu.wav invalid.wav choice ^[1-3]$"/>
            <action application="transfer" data="option_${choice} XML authenticated"/>
        </condition>
    </extension>
</context>

<context name="authenticated">
    <extension name="validate">
        <!-- Require authentication before proceeding -->
        <condition field="${authenticated}" expression="^true$" break="on-false">
            <action application="transfer" data="process_authenticated XML internal"/>
            <anti-action application="log" data="ERR Unauthorized access attempt"/>
            <anti-action application="hangup" data="CALL_REJECTED"/>
        </condition>
    </extension>
</context>
```

### Pattern 2: Multi-Tenant Architecture

```
tenant_router
  ├─→ tenant_a_context
  │    ├─→ tenant_a_ivr
  │    └─→ tenant_a_internal
  └─→ tenant_b_context
       ├─→ tenant_b_ivr
       └─→ tenant_b_internal
```

**Implementation:**
```xml
<context name="tenant_router">
    <extension name="route_by_domain">
        <condition field="${domain_name}" expression="^tenant-a\.com$">
            <action application="transfer" data="${destination_number} XML tenant_a_context"/>
        </condition>
        <condition field="${domain_name}" expression="^tenant-b\.com$">
            <action application="transfer" data="${destination_number} XML tenant_b_context"/>
        </condition>
    </extension>
</context>

<context name="tenant_a_context">
    <!-- Completely isolated from tenant_b -->
    <extension name="tenant_a_extensions">
        <condition field="destination_number" expression="^(\d{4})$">
            <action application="bridge" data="user/$1@tenant-a.com"/>
        </condition>
    </extension>
</context>
```

### Pattern 3: Call Processing Pipeline

```
entry
  ↓
validation
  ↓
routing
  ↓
execution
```

**Implementation:**
```xml
<context name="entry">
    <extension name="capture_info">
        <condition field="destination_number" expression="^(.+)$">
            <action application="set" data="call_source=external"/>
            <action application="set" data="call_timestamp=${strftime(%Y%m%d%H%M%S)}"/>
            <action application="transfer" data="validate XML validation"/>
        </condition>
    </extension>
</context>

<context name="validation">
    <extension name="check_requirements">
        <condition field="${sip_from_host}" expression="^$" break="on-true">
            <action application="log" data="ERR Invalid SIP From header"/>
            <action application="hangup" data="INVALID_MSG_UNSPECIFIED"/>
        </condition>
        <condition field="destination_number" expression="^(.+)$">
            <action application="transfer" data="$1 XML routing"/>
        </condition>
    </extension>
</context>

<context name="routing">
    <!-- Routing logic here -->
</context>
```

### Pattern 4: Service Layer Architecture

```
main_context
  ├─→ services (reusable functions)
  │    ├─→ check_business_hours
  │    ├─→ validate_caller
  │    └─→ log_call
  └─→ features
       ├─→ voicemail
       ├─→ conference
       └─→ call_forwarding
```

**Implementation:**
```xml
<context name="main_context">
    <extension name="service_call">
        <condition field="destination_number" expression="^1000$">
            <action application="set" data="return_context=main_context"/>
            <action application="set" data="return_extension=continue_processing"/>
            <action application="execute_extension" data="check_business_hours XML services"/>
        </condition>
    </extension>

    <extension name="continue_processing">
        <condition field="${business_hours}" expression="^open$">
            <action application="bridge" data="user/1000@${domain}"/>
            <anti-action application="transfer" data="voicemail XML features"/>
        </condition>
    </extension>
</context>

<context name="services">
    <extension name="check_business_hours">
        <condition wday="2-6" time-of-day="08:00-17:00">
            <action application="set" data="business_hours=open"/>
            <anti-action application="set" data="business_hours=closed"/>
        </condition>
        <!-- Return to caller -->
        <action application="transfer" data="${return_extension} XML ${return_context}"/>
    </extension>
</context>
```

---

## Best Practices for Context Design

### 1. Use Descriptive Context Names

**Good:**
```xml
<context name="public_pstn_inbound"/>
<context name="authenticated_internal"/>
<context name="emergency_services"/>
<context name="ivr_main_menu"/>
```

**Bad:**
```xml
<context name="context1"/>
<context name="test"/>
<context name="stuff"/>
```

### 2. Document Context Purpose

```xml
<!--
  Context: public_pstn_inbound
  Purpose: Entry point for all inbound PSTN calls
  Trust Level: Untrusted
  Access: Limited to published DIDs only
  Next Stages: ivr_main_menu, authenticated_internal
-->
<context name="public_pstn_inbound">
    <!-- extensions -->
</context>
```

### 3. Validate Context Entry Points

**Always validate when entering critical contexts:**

```xml
<context name="internal_extensions">
    <!-- First extension: validation -->
    <extension name="validate_authentication">
        <condition field="${authenticated}" expression="^$" break="on-true">
            <action application="log" data="ERR Unauthorized access to internal context"/>
            <action application="hangup" data="CALL_REJECTED"/>
        </condition>
    </extension>

    <!-- Subsequent extensions: actual logic -->
    <extension name="internal_routing">
        <!-- Now we know caller is authenticated -->
    </extension>
</context>
```

### 4. Keep Context Dependencies Minimal

**Bad: Context B depends on Context A:**
```xml
<context name="context_a">
    <extension name="set_stuff">
        <action application="set" data="important_data=value"/>
        <action application="transfer" data="process XML context_b"/>
    </extension>
</context>

<context name="context_b">
    <!-- Silently depends on important_data being set -->
    <extension name="process">
        <action application="log" data="INFO ${important_data}"/>
    </extension>
</context>
```

**Good: Context B validates its requirements:**
```xml
<context name="context_b">
    <extension name="validate_and_process">
        <!-- Explicit validation -->
        <condition field="${important_data}" expression="^$" break="on-true">
            <action application="log" data="ERR Missing required data"/>
            <action application="hangup"/>
        </condition>

        <!-- Now safe to use -->
        <condition field="destination_number" expression="^process$">
            <action application="log" data="INFO ${important_data}"/>
        </condition>
    </extension>
</context>
```

### 5. Use Global Extensions Sparingly

**Global extensions (continue="true") should:**
- Set universal defaults
- Never do complex logic
- Never transfer or bridge
- Be at the TOP of the context

```xml
<context name="internal">
    <!-- Global settings first -->
    <extension name="global" continue="true">
        <condition break="never">
            <action application="set" data="hangup_after_bridge=true"/>
            <action application="set" data="continue_on_fail=true"/>
        </condition>
    </extension>

    <!-- Then feature extensions -->
    <extension name="local_extension">
        <!-- actual routing -->
    </extension>
</context>
```

### 6. Emergency Services Get Their Own Context

**Always isolate emergency routing:**

```xml
<context name="emergency_services">
    <!-- No other logic, just emergency routing -->
    <extension name="911">
        <condition field="destination_number" expression="^(911|112|999)$">
            <action application="set" data="emergency_call=true"/>
            <action application="set" data="effective_caller_id_number=${caller_id_number}"/>
            <action application="bridge" data="sofia/gateway/emergency_gateway/$1"/>
        </condition>
    </extension>
</context>
```

### 7. Don't Mix Trust Levels

**Bad: Public and internal in same context**
```xml
<context name="mixed">
    <extension name="external_did">
        <!-- External untrusted calls -->
    </extension>
    <extension name="internal_extension">
        <!-- Internal trusted calls -->
    </extension>
</context>
```

**Good: Separate contexts**
```xml
<context name="public">
    <!-- Only external DIDs -->
</context>

<context name="internal">
    <!-- Only internal extensions -->
</context>
```

---

## Common Anti-Patterns

### Anti-Pattern 1: God Context

**Problem:** One massive context that does everything

```xml
<context name="default">
    <extension name="external_calls">...</extension>
    <extension name="internal_calls">...</extension>
    <extension name="ivr">...</extension>
    <extension name="emergency">...</extension>
    <extension name="admin">...</extension>
    <!-- 50 more extensions... -->
</context>
```

**Solution:** Split by function and trust level

### Anti-Pattern 2: Context Chains

**Problem:** Calls bounce through many contexts unnecessarily

```xml
context_a → context_b → context_c → context_d → context_e
```

**Solution:** Design direct paths with 2-3 stages maximum

### Anti-Pattern 3: Duplicate Logic

**Problem:** Same logic repeated in multiple contexts

**Solution:** Use execute_extension to call shared logic, or use global extension with continue="true"

### Anti-Pattern 4: No Validation

**Problem:** Contexts assume variables exist

**Solution:** Always validate at context entry

### Anti-Pattern 5: Tight Coupling

**Problem:** Contexts that can't work independently

**Solution:** Each context should be self-contained with clear interfaces

---

## Real-World Examples

### Example 1: Call Center with Security Layers

```xml
<!-- Entry point: External PSTN -->
<context name="public_pstn">
    <extension name="main_line">
        <condition field="destination_number" expression="^8005551234$">
            <action application="set" data="call_source=pstn"/>
            <action application="set" data="call_entry_time=${strftime(%s)}"/>
            <action application="answer"/>
            <action application="transfer" data="language_select XML ivr"/>
        </condition>
    </extension>
</context>

<!-- IVR Layer: Semi-trusted -->
<context name="ivr">
    <extension name="language_select">
        <condition field="destination_number" expression="^language_select$">
            <action application="play_and_get_digits" data="1 1 3 5000 # lang.wav invalid.wav lang ^[12]$"/>
            <action application="transfer" data="main_menu_${lang} XML ivr_menus"/>
        </condition>
    </extension>
</context>

<!-- IVR Menus: Controlled options -->
<context name="ivr_menus">
    <extension name="main_menu_1">
        <condition field="destination_number" expression="^main_menu_1$">
            <action application="play_and_get_digits" data="1 1 3 5000 # menu_en.wav invalid.wav option ^[1-4]$"/>
            <action application="transfer" data="queue_${option} XML queue_routing"/>
        </condition>
    </extension>
</context>

<!-- Queue Routing: Validated routing -->
<context name="queue_routing">
    <extension name="validate_option">
        <condition field="${option}" expression="^$" break="on-true">
            <action application="log" data="ERR Invalid queue option"/>
            <action application="transfer" data="main_menu_1 XML ivr_menus"/>
        </condition>
    </extension>

    <extension name="queue_1">
        <condition field="destination_number" expression="^queue_1$">
            <action application="set" data="queue_name=sales"/>
            <action application="transfer" data="process_queue XML queue_handler"/>
        </condition>
    </extension>
</context>

<!-- Queue Handler: Internal operations -->
<context name="queue_handler">
    <extension name="process_queue">
        <condition field="destination_number" expression="^process_queue$">
            <action application="set" data="fifo_music=$${hold_music}"/>
            <action application="fifo" data="${queue_name} in"/>
        </condition>
    </extension>
</context>
```

### Example 2: Multi-Tenant with Emergency

```xml
<!-- Router: Determine tenant -->
<context name="multi_tenant_router">
    <extension name="route_by_domain">
        <condition field="${domain_name}" expression="^tenant1\.com$" break="on-true">
            <action application="set" data="tenant_id=tenant1"/>
            <action application="transfer" data="${destination_number} XML tenant1_main"/>
        </condition>
        <condition field="${domain_name}" expression="^tenant2\.com$" break="on-true">
            <action application="set" data="tenant_id=tenant2"/>
            <action application="transfer" data="${destination_number} XML tenant2_main"/>
        </condition>
    </extension>
</context>

<!-- Tenant 1: Isolated context -->
<context name="tenant1_main">
    <!-- Emergency always works -->
    <extension name="emergency">
        <condition field="destination_number" expression="^(911|112)$" break="on-true">
            <action application="transfer" data="emergency_${tenant_id} XML emergency_services"/>
        </condition>
    </extension>

    <!-- Tenant-specific extensions -->
    <extension name="tenant1_local">
        <condition field="destination_number" expression="^(\d{4})$">
            <action application="bridge" data="user/$1@tenant1.com"/>
        </condition>
    </extension>
</context>

<!-- Emergency Services: Shared but logged by tenant -->
<context name="emergency_services">
    <extension name="emergency_tenant1">
        <condition field="destination_number" expression="^emergency_tenant1$">
            <action application="set" data="emergency_tenant=tenant1"/>
            <action application="log" data="ALERT Emergency call from tenant1: ${caller_id_number}"/>
            <action application="bridge" data="sofia/gateway/emergency_gw/911"/>
        </condition>
    </extension>
</context>
```

---

## Agent Decision Tree

When helping a user design a dialplan, use this decision tree:

```
Is there a mix of:
├─ Emergency + Regular calls?
│  └─ YES → Separate contexts (emergency_services, regular_routing)
├─ External + Internal sources?
│  └─ YES → Separate contexts (public, internal)
├─ Different trust levels?
│  └─ YES → Separate contexts by trust level
├─ Multi-stage workflow? (IVR → Queue → Agent)
│  └─ YES → Separate contexts per stage
├─ Multi-tenant?
│  └─ YES → Separate contexts per tenant
└─ Simple routing variations?
   └─ YES → Same context, multiple extensions

For each context boundary:
├─ Add validation extension at entry
├─ Document trust level
├─ Minimize variable dependencies
└─ Ensure independent operation
```

---

## Summary: Your Role as Dialplan Architect

When collaborating with users on designing a dialplan,

1. **Ask about security boundaries** - External vs internal? Authenticated vs anonymous?

2. **Identify mixing scenarios** - When you see:
   - Emergency + regular calls
   - Public PSTN + internal extensions
   - IVR + direct dialing
   - Workstations + trunks
   → **Suggest separate contexts**

3. **Validate context isolation** - Each context should:
   - Have clear purpose
   - Validate entry requirements
   - Work independently
   - Not assume variables exist

4. **Recommend appropriate transfers** - Use:
   - `transfer` for security boundaries and stage changes
   - `execute_extension` for subroutines and utilities

5. **Check for anti-patterns**:
   - God contexts (too much in one place)
   - Context chains (too many hops)
   - Missing validation (no entry checks)
   - Tight coupling (hidden dependencies)

6. **Document your recommendations** - Explain:
   - Why separate contexts
   - What each context does
   - How they interact
   - What variables are required

**Remember:** Contexts are your primary tool for creating secure, maintainable, and well-organized dialplans. Use them deliberately and explain your reasoning to users.
