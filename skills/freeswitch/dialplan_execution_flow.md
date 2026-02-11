# FreeSWITCH Dialplan Execution Flow

## Understanding the Two-Level Structure

FreeSWITCH dialplans have a two-level nested structure that many people misunderstand:

```
Context
  └─ Extension (level 1)
       └─ Condition (level 2)
            └─ Action/Anti-Action
```

## The Two Control Attributes

### 1. `continue` Attribute (on `<extension>`)
**Controls:** Whether to process the **next extension** after this one matches

### 2. `break` Attribute (on `<condition>`)
**Controls:** Whether to process the **next condition** within the same extension

These are **independent** and control different loops!

## Source Code Reference

From `mod_dialplan_xml.c`:

### Extension Loop (Outer Loop)
```c
// Line 682-708: Loop through extensions
while (xexten) {
    const char *cont = switch_xml_attr(xexten, "continue");

    proceed = parse_exten(session, caller_profile, xexten, &extension, exten_name, 0);

    if (proceed && !switch_true(cont)) {
        break;  // Stop processing more extensions
    }

    xexten = xexten->next;  // Move to next extension
}
```

### Condition Loop (Inner Loop)
```c
// Line 167: Loop through conditions within an extension
for (xcond = switch_xml_child(xexten, "condition"); xcond; xcond = xcond->next) {
    // ... process condition ...

    // Line 576-579: Break logic
    if (((anti_action == SWITCH_FALSE && do_break_i == BREAK_ON_TRUE) ||
         (anti_action == SWITCH_TRUE && do_break_i == BREAK_ON_FALSE)) ||
         do_break_i == BREAK_ALWAYS) {
        break;  // Stop processing more conditions in this extension
    }
}
```

## Extension-Level Control: `continue` Attribute

**Location:** On the `<extension>` element
**Default:** `continue="false"` (stop after first match)

### Behavior

```xml
<extension name="first">
    <condition field="destination_number" expression="^1000$">
        <action application="set" data="matched_first=true"/>
    </condition>
</extension>

<extension name="second">
    <condition field="destination_number" expression="^1000$">
        <action application="set" data="matched_second=true"/>
    </condition>
</extension>
```

**Result:** Only `first` extension processes. `second` is never evaluated.

### With `continue="true"`

```xml
<extension name="first" continue="true">
    <condition field="destination_number" expression="^1000$">
        <action application="set" data="matched_first=true"/>
    </condition>
</extension>

<extension name="second">
    <condition field="destination_number" expression="^1000$">
        <action application="set" data="matched_second=true"/>
    </condition>
</extension>
```

**Result:** Both `first` AND `second` extensions process. Both variables get set.

### Common Use Case: Global Settings

```xml
<extension name="global" continue="true">
    <condition break="never">
        <action application="set" data="domain=${domain_name}"/>
        <action application="set" data="hangup_after_bridge=true"/>
    </condition>
</extension>

<!-- This extension will ALSO process because global has continue="true" -->
<extension name="local_extension">
    <condition field="destination_number" expression="^(\d{4})$">
        <action application="bridge" data="user/$1@${domain}"/>
    </condition>
</extension>
```

## Condition-Level Control: `break` Attribute

**Location:** On the `<condition>` element
**Default:** `break="on-false"` (stop if condition fails)

### Break Values

| Value | Behavior |
|-------|----------|
| `break="on-true"` | Stop processing conditions if this condition **matches** |
| `break="on-false"` | Stop processing conditions if this condition **doesn't match** (default) |
| `break="always"` | Stop processing conditions regardless of match |
| `break="never"` | Continue processing conditions regardless of match |

### Example: `break="on-false"` (Default)

```xml
<extension name="example">
    <!-- Condition 1: Check caller -->
    <condition field="caller_id_number" expression="^1000$">
        <!-- This matches, continue to next condition -->
    </condition>

    <!-- Condition 2: Check destination -->
    <condition field="destination_number" expression="^2000$">
        <action application="bridge" data="user/2000"/>
    </condition>
</extension>
```

**If caller is 1000 and destination is 2000:** Both conditions match, actions execute.
**If caller is NOT 1000:** First condition fails, `break="on-false"` stops, second condition never evaluated.

### Example: `break="never"` (Always Continue)

```xml
<extension name="global" continue="true">
    <condition break="never">
        <!-- This always executes, even if no field/expression -->
        <action application="set" data="my_var=value"/>
    </condition>
</extension>
```

This is the standard pattern for "global" extensions that set variables for all calls.

### Example: `break="on-true"` (Stop When Matches)

```xml
<extension name="check_business_hours">
    <!-- Check if closed -->
    <condition wday="1,7" break="on-true">
        <!-- Weekend - stop checking, go to voicemail -->
        <action application="voicemail" data="default ${domain} ${destination_number}"/>
    </condition>

    <!-- This only evaluates if NOT weekend -->
    <condition field="destination_number" expression="^(\d{4})$">
        <action application="bridge" data="user/$1@${domain}"/>
    </condition>
</extension>
```

## How They Work Together

### Example: Multiple Extensions with Multiple Conditions

```xml
<extension name="set_variables" continue="true">
    <condition break="never">
        <action application="set" data="ringback=${us-ring}"/>
    </condition>
    <condition break="never">
        <action application="set" data="transfer_ringback=${hold_music}"/>
    </condition>
</extension>

<extension name="check_access" continue="true">
    <condition field="caller_id_number" expression="^blocked$" break="on-true">
        <action application="hangup" data="CALL_REJECTED"/>
    </condition>
    <condition field="caller_id_number" expression="^allowed$">
        <action application="set" data="authorized=true"/>
    </condition>
</extension>

<extension name="route_call">
    <condition field="destination_number" expression="^(\d{4})$">
        <action application="bridge" data="user/$1@${domain}"/>
    </condition>
</extension>
```

**Execution flow:**
1. **Extension "set_variables"** processes:
   - Condition 1 (break="never"): Sets ringback
   - Condition 2 (break="never"): Sets transfer_ringback
   - Has `continue="true"`, so continue to next extension

2. **Extension "check_access"** processes:
   - Condition 1: Check if blocked
     - If blocked: hangup, `break="on-true"` stops conditions, but `continue="true"` still processes next extension
     - If not blocked: continue to next condition
   - Condition 2: Check if allowed
   - Has `continue="true"`, so continue to next extension

3. **Extension "route_call"** processes:
   - Condition 1: Route the call
   - No `continue`, so stop after this extension

## Nested Conditions

When you have nested conditions, the `break` attribute of the inner condition only affects its own level:

```xml
<condition field="${outer}" expression="^pattern$" break="on-true">
    <action application="set" data="outer_matched=true"/>

    <!-- Inner condition loop -->
    <condition field="${inner}" expression="^value$" break="on-true">
        <action application="set" data="inner_matched=true"/>
    </condition>

    <!-- This condition is at the same (inner) level -->
    <condition field="${inner2}" expression="^value2$">
        <action application="set" data="inner2_matched=true"/>
    </condition>
</condition>
```

The `break="on-true"` on the outer condition affects the main condition loop of the extension. The `break="on-true"` on the inner condition only affects the nested condition loop.

## Common Patterns

### Pattern 1: Global Settings That Always Run
```xml
<extension name="global" continue="true">
    <condition break="never">
        <action application="set" data="variable=value"/>
    </condition>
</extension>
```

### Pattern 2: Information Extraction That Always Runs
```xml
<extension name="extract_info" continue="true">
    <condition field="${sip_h_X-Custom}" expression="(.+)">
        <action application="set" data="extracted=$1" inline="true"/>
    </condition>
</extension>
```

### Pattern 3: Sequential Checks That Stop on First Match
```xml
<extension name="routing">
    <condition field="destination_number" expression="^911$" break="on-true">
        <action application="bridge" data="emergency"/>
    </condition>
    <condition field="destination_number" expression="^(\d{4})$" break="on-true">
        <action application="bridge" data="user/$1"/>
    </condition>
    <condition>
        <action application="hangup" data="NO_ROUTE_DESTINATION"/>
    </condition>
</extension>
```

### Pattern 4: Validation That Stops on Failure
```xml
<extension name="validate_and_route">
    <condition field="${required_var}" expression="^$" break="on-true">
        <action application="log" data="ERR Missing required variable"/>
        <action application="hangup" data="NORMAL_UNSPECIFIED"/>
    </condition>

    <!-- Only executes if required_var exists -->
    <condition field="destination_number" expression="^(\d+)$">
        <action application="bridge" data="destination/$1"/>
    </condition>
</extension>
```

## Summary

**Remember:**
- `continue` on `<extension>` → Controls processing of **next extension**
- `break` on `<condition>` → Controls processing of **next condition within same extension**

These operate at different levels and are independent of each other. Understanding this two-level structure is crucial to writing correct FreeSWITCH dialplans.
