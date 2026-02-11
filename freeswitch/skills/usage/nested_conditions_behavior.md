# FreeSWITCH Nested Conditions: How They Really Work

## Source Code Analysis
Based on examination of `/src/mod/dialplans/mod_dialplan_xml/mod_dialplan_xml.c`

## Key Code References

### 1. Condition Loop (Line 167)
```c
for (xcond = switch_xml_child(xexten, "condition"); xcond; xcond = xcond->next) {
```

### 2. Break Attribute Processing (Lines 187-199)
```c
if ((do_break_a = (char *) switch_xml_attr(xcond, "break"))) {
    if (!strcasecmp(do_break_a, "on-true")) {
        do_break_i = BREAK_ON_TRUE;
    } else if (!strcasecmp(do_break_a, "on-false")) {
        do_break_i = BREAK_ON_FALSE;
    } else if (!strcasecmp(do_break_a, "always")) {
        do_break_i = BREAK_ALWAYS;
    } else if (!strcasecmp(do_break_a, "never")) {
        do_break_i = BREAK_NEVER;
    }
}
```

### 3. Action Processing (Lines 512-572)
When a condition MATCHES (`anti_action == SWITCH_FALSE`):
```c
for (xaction = switch_xml_child(xcond, "action"); xaction; xaction = xaction->next) {
}
```

### 4. Anti-Action Processing (Lines 461-505)
When a condition FAILS (`anti_action == SWITCH_TRUE`):
```c
for (xaction = switch_xml_child(xcond, "anti-action"); xaction; xaction = xaction->next) {
}
```

### 5. Break Logic (Lines 576-579)
```c
if (((anti_action == SWITCH_FALSE && do_break_i == BREAK_ON_TRUE) ||
     (anti_action == SWITCH_TRUE && do_break_i == BREAK_ON_FALSE)) ||
     do_break_i == BREAK_ALWAYS) {
    break;
}
```

### 6. Nested Condition Processing (Lines 581-590)
```c
if (proceed) {
    if (switch_xml_child(xcond, "condition")) {
        if (!(proceed = parse_exten(session, caller_profile, xcond, extension,
                                   orig_exten_name, recur + 1))) {
            if (do_break_i == BREAK_NEVER) {
                continue;
            }
            goto done;
        }
    }
}
```

## Execution Flow Example

Given this dialplan:
```xml
<condition field="${sip_h_X-destination_number}" expression="^\+?(18005551234)$" break="on-true">
    <action application="set" data="dialed_number=$1"/>
    <condition field="${hostname}" expression="^server[1-9]$" break="on-true">
        <action application="log" data="INFO - Bridging to pbx"/>
        <action application="bridge" data="sofia/gateway/${pbx_ic_1}/${dialed_number}"/>
        <anti-action application="log" data="INFO - Bridging to server"/>
        <anti-action application="bridge" data="sofia/gateway/${sbc_ic_1}/${dialed_number}|sofia/gateway/${sbc_ic_2}/${dialed_number}"/>
    </condition>
</condition>
```

### When sip_h_X-destination_number = "18005551234"

#### Step 1: Outer Condition Evaluation
- Field: `${sip_h_X-destination_number}` = "18005551234"
- Pattern: `^\+?(18005551234)$`
- **MATCHES** ✓
- Sets: `anti_action = SWITCH_FALSE` (condition passed)
- Captures: `$1 = "18005551234"`

#### Step 2: Process Outer Condition's Direct Actions (Lines 512-572)
```xml
<action application="set" data="dialed_number=$1"/>
```
- Sets: `dialed_number = "18005551234"`

#### Step 3: Check for Nested Conditions (Lines 581-590)
- Finds nested `<condition>` element
- Recursively calls `parse_exten` with the nested condition

#### Step 4: Nested Condition Evaluation
- Field: `${hostname}`
- Pattern: `^server[1-9]$`

**Scenario A**: hostname = "server1" (matches)
- Sets: `anti_action = SWITCH_FALSE`
- Executes nested actions:
  - `log` "INFO - Bridging to pbx"
  - `bridge` to `sofia/gateway/${pbx_ic_1}/18005551234`
- Nested `break="on-true"` causes recursive function to return

**Scenario B**: hostname = "server2" (doesn't match)
- Sets: `anti_action = SWITCH_TRUE`
- Executes nested anti-actions:
  - `log` "INFO - Bridging to server"
  - `bridge` to failover gateways

#### Step 5: Return from Recursion
- After nested condition completes, returns to outer condition
- Outer condition's `break="on-true"` now takes effect
- Stops processing further conditions in the extension

## Double break="on-true" Behavior

When you have nested conditions both with `break="on-true"`:

- **Outer break**: Stops processing further condition siblings (after nested conditions complete)
- **Inner break**: Exits the recursive call (doesn't affect outer condition processing)

## Critical Insights

1. **Break timing is crucial**: Outer break happens AFTER nested processing completes; inner break only affects the recursive call return

2. **Order of operations**:
   1. Evaluate outer condition
   2. Execute ALL direct actions/anti-actions
   3. Process nested conditions
   4. Apply outer condition's break logic
   5. Continue to next sibling (unless break stops it)

## Anti-Pattern: Unnecessary Nesting That Confuses Debugging

### The Problem

When nested conditions are used unnecessarily (not for actual conditional logic), they cause confusing log output that makes debugging difficult.

### Bad Example: Data Extraction in Nested Conditions

```xml
<extension name="ngcs_create_conference">
    <condition field="destination_number" expression="^(ngcs_create_conference)$">
        <condition>
            <action application="set" data="oreka_sip_h_CallUniqueIdentifier=${sip_h_X-C911P-Unique-ID}" inline="true"/>
        </condition>

        <condition field="${sip_h_X-Call-Info}" expression="urn:emergency:uid:incidentid:([\w\d]+)" break="never">
            <action application="set" data="oreka_sip_h_Incident ID=$1" inline="true"/>
        </condition>

        <condition field="${sip_h_X-Call-Info}" expression="urn:emergency:uid:callid:([\w\d]+)" break="never">
            <action application="set" data="oreka_sip_h_Call ID=$1" inline="true"/>
        </condition>

        <action application="export" data="nolocal:sip_invite_call_id=${sip_call_id}"/>
        <action application="set_profile_var" data="callee_id_number=${caller_id_name}"/>
        <action application="oreka_record"/>
        <action application="bridge" data="sofia/gateway/primary/$1"/>
    </condition>
</extension>
```

### Log Output Confusion

The logs show nested conditions executing **during parsing phase** with inline actions, then outer actions executing **later in execute phase**:

```
|--- Dialplan: Processing recursive conditions level:1 [ngcs_create_conference_recur_1]
|--- EXECUTE [depth=0] sofia/internal/1233... set(oreka_sip_h_CallUniqueIdentifier=3fd9e5b8...)
|--- EXECUTE [depth=0] sofia/internal/1233... set(oreka_sip_h_Incident ID=20251110221508574e1p...)
|--- EXECUTE [depth=0] sofia/internal/1233... set(oreka_sip_h_Call ID=20251110221508574LU1...)
|--- State Change CS_ROUTING -> CS_EXECUTE
EXECUTE [depth=0] sofia/internal/1233... export(nolocal:sip_invite_call_id=...)
EXECUTE [depth=0] sofia/internal/1233... set_profile_var(callee_id_number=...)
EXECUTE [depth=0] sofia/internal/1233... oreka_record()
EXECUTE [depth=0] sofia/internal/1233... bridge(...)
```

**Problems:**
1. Actions appear in **non-sequential order** in logs (nested inline first, then outer actions)
2. **"Processing recursive conditions"** message suggests complex logic where there is none
3. **Depth markers** (`|---`) make logs harder to grep and read
4. Inline execution during parsing phase is **separated** from execute phase actions
5. Difficult to trace actual execution order when troubleshooting

### Better: Flatten to Sequential Conditions

```xml
<extension name="ngcs_create_conference">
    <condition field="destination_number" expression="^(ngcs_create_conference)$" break="on-false"/>

    <condition break="never">
        <action application="set" data="oreka_sip_h_CallUniqueIdentifier=${sip_h_X-C911P-Unique-ID}" inline="true"/>
    </condition>

    <condition field="${sip_h_X-Call-Info}" expression="urn:emergency:uid:incidentid:([\w\d]+)" break="never">
        <action application="set" data="oreka_sip_h_Incident ID=$1" inline="true"/>
    </condition>

    <condition field="${sip_h_X-Call-Info}" expression="urn:emergency:uid:callid:([\w\d]+)" break="never">
        <action application="set" data="oreka_sip_h_Call ID=$1" inline="true"/>
    </condition>

    <condition>
        <action application="export" data="nolocal:sip_invite_call_id=${sip_call_id}"/>
        <action application="set_profile_var" data="callee_id_number=${caller_id_name}"/>
        <action application="oreka_record"/>
        <action application="bridge" data="sofia/gateway/primary/$1"/>
    </condition>
</extension>
```

### Clearer Log Output

With flattened conditions, logs show **sequential execution** in order:

```
EXECUTE [depth=0] sofia/internal/1233... set(oreka_sip_h_CallUniqueIdentifier=3fd9e5b8...)
EXECUTE [depth=0] sofia/internal/1233... set(oreka_sip_h_Incident ID=20251110221508574e1p...)
EXECUTE [depth=0] sofia/internal/1233... set(oreka_sip_h_Call ID=20251110221508574LU1...)
EXECUTE [depth=0] sofia/internal/1233... export(nolocal:sip_invite_call_id=...)
EXECUTE [depth=0] sofia/internal/1233... set_profile_var(callee_id_number=...)
EXECUTE [depth=0] sofia/internal/1233... oreka_record()
EXECUTE [depth=0] sofia/internal/1233... bridge(...)
```

**Benefits:**
- Actions appear in **actual execution order**
- No confusing recursion markers
- Easy to grep and filter logs
- Clear progression from setup → execute → bridge
- Debugging is straightforward

### Rule of Thumb

**Only use nested conditions when:**
- You need actual conditional branching based on parent condition results
- Parent actions must execute BEFORE evaluating child conditions
- The nesting represents real logical dependency

**Don't use nested conditions for:**
- Simple data extraction that always runs
- Sequential operations
- Variable setup before bridge/transfer
- Anything that could be flattened to same-level conditions with `break="never"`