# FreeSWITCH XML Dialplan Execution Flow - Definitive Guide

## Overview

FreeSWITCH dialplan processing consists of two distinct phases:
1. **Planning Phase (Hunt Phase)**: Condition evaluation, action stacking, inline execution
2. **Execution Phase**: Sequential execution of all stacked (non-inline) actions

## Code References

Primary implementation: `/src/mod/dialplans/mod_dialplan_xml/mod_dialplan_xml.c`

Key functions:
- `dialplan_hunt()` (line 620): Entry point for dialplan processing
- `parse_exten()` (line 98): Recursive condition/action processor
- `exec_app()` (line 48): Inline action executor

State machine: `/src/switch_core_state_machine.c`
- `switch_core_standard_on_routing()`: Calls dialplan hunt, transitions to CS_EXECUTE
- `switch_core_standard_on_execute()`: Executes stacked applications

## Phase 1: Planning/Hunt Phase

### Entry Point
**File**: `mod_dialplan_xml.c`
**Function**: `dialplan_hunt()` (line 620)
**Trigger**: Channel state `CS_ROUTING`

### Process Flow

```
dialplan_hunt()
  → loops through extensions (line 682)
    → calls parse_exten() for each extension (line 701)
      → loops through conditions (line 167)
        → evaluates condition regex/field (lines 409-443)
        → if condition PASSES (anti_action = FALSE):
          → captures regex groups (lines 507-509)
          → loops through <action> children (line 512)
            → checks inline="true" attribute (lines 519-520)
            → if inline=true:
              → exec_app() IMMEDIATELY (line 566)
            → if inline=false or not set:
              → switch_caller_extension_add_application() STACKS action (line 568)
          → checks for nested <condition> children (line 582)
            → if nested conditions exist:
              → RECURSIVE call to parse_exten() (line 583)
        → evaluates break attribute (lines 576-578)
      → checks continue="true" on extension (line 703)
  → returns extension with stacked actions
```

### Critical Code Sections

#### Action Processing (lines 512-573)
```c
for (xaction = switch_xml_child(xcond, "action"); xaction; xaction = xaction->next) {
    const char *inline_ = switch_xml_attr_soft(xaction, "inline");
    int xinline = switch_true(inline_);

    if (xinline) {
        exec_app(session, application, app_data);  // EXECUTES IMMEDIATELY
    } else {
        switch_caller_extension_add_application(session, *extension, application, app_data);  // STACKS
    }
}
```

#### Nested Condition Recursion (lines 581-590)
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

**CRITICAL**: Nested conditions are evaluated DURING the planning phase, AFTER parent actions are processed.

### Planning Phase Order

For a parent condition with nested conditions:

1. **Parent condition evaluation** (line 409)
2. **Parent regex captures** (lines 507-509)
3. **Parent action processing loop** (lines 512-573):
   - Inline actions execute immediately
   - Non-inline actions are stacked in order
4. **Check for nested conditions** (line 582)
5. **Recursive parse_exten() for nested conditions** (line 583)
   - Nested conditions evaluated
   - Nested actions processed (inline executed, others stacked)
6. **Break logic evaluation** (lines 576-578)

## Phase 2: Execution Phase

### Entry Point
**File**: `switch_core_state_machine.c`
**Function**: `switch_core_standard_on_execute()`
**Trigger**: Channel state `CS_EXECUTE` (set at line 701 of routing phase)

### Process Flow

```c
// Simplified from switch_core_state_machine.c
if ((extension = switch_channel_get_caller_extension(session->channel)) == 0) {
    switch_channel_hangup(session->channel, SWITCH_CAUSE_NORMAL_CLEARING);
    return;
}

while (switch_channel_get_state(session->channel) == CS_EXECUTE &&
       extension->current_application) {
    switch_caller_application_t *current_application = extension->current_application;

    extension->current_application = extension->current_application->next;

    switch_core_session_execute_application(session,
                                           current_application->application_name,
                                           current_application->application_data);
}
```

### Execution Order

Actions execute in the **exact order they were stacked** during planning phase:

1. Parent condition's first action
2. Parent condition's second action
3. ...
4. Nested condition's first action
5. Nested condition's second action
6. ...

All actions execute **sequentially** with blocking behavior.

## Critical Example Analysis

### Example Dialplan
```xml
<condition field="${var}" expression="^match$" break="on-true">
    <action application="set" data="outer_var=value"/>  <!-- NO inline -->
    <condition field="${hostname}" expression="^server$" break="on-true">
        <action application="bridge" data="destination"/>
    </condition>
</condition>
```

### Execution Trace

#### Planning Phase:
```
1. Evaluate parent condition: ${var} =~ /^match$/
   → Result: PASS (proceed = 1)

2. Set regex captures for parent condition (DP_MATCH)

3. Process parent actions:
   → action: set(outer_var=value)
   → inline = false
   → STACK action (extension->applications list)
   → Log: "Dialplan: Action set(outer_var=value)"

4. Check for nested conditions:
   → Found: <condition field="${hostname}">

5. RECURSIVE call to parse_exten() for nested condition:
   5a. Evaluate nested condition: ${hostname} =~ /^server$/
       → Result: PASS

   5b. Set regex captures for nested condition

   5c. Process nested actions:
       → action: bridge(destination)
       → inline = false
       → STACK action (extension->applications list)
       → Log: "Dialplan: Action bridge(destination)"

   5d. Check nested condition's break="on-true"
       → Breaks from nested condition loop

6. Check parent condition's break="on-true"
   → Breaks from parent condition loop

7. Return extension with 2 stacked actions
```

#### Execution Phase:
```
1. Channel state → CS_EXECUTE

2. Get extension->current_application (first action)
   → Execute: set(outer_var=value)
   → Variable outer_var is SET NOW

3. Get extension->current_application (second action)
   → Execute: bridge(destination)
   → Bridge blocks until call completes
```

## Answer to Specific Questions

### Q1: Are conditions evaluated during planning?
**YES** - All conditions are evaluated during the planning phase via `switch_regex_perform()` at line 409.

### Q2: Do inline="true" actions execute immediately during planning?
**YES** - Line 566 shows `exec_app(session, application, app_data)` is called immediately when `xinline = true`.

### Q3: Are regular actions (without inline) STACKED but not executed during planning?
**YES** - Line 568 shows `switch_caller_extension_add_application()` only adds to a list. No execution occurs.

### Q4: Are nested conditions evaluated during planning phase?
**YES** - Line 583 shows recursive `parse_exten()` call happens during parent's planning phase.

### Q5: In the critical example, does the outer set action execute BEFORE nested condition evaluation?
**NO** - The answer is **B): The set action is STACKED during planning**.

#### Detailed Answer for Critical Example:

**During Planning Phase:**
1. Parent condition evaluates → PASS
2. Parent `set` action → **STACKED** (not executed)
3. Nested condition evaluates → PASS (occurs BEFORE set executes)
4. Nested `bridge` action → **STACKED**

**During Execution Phase:**
1. `set(outer_var=value)` executes → Variable set NOW
2. `bridge(destination)` executes → Uses the variable

**Critical Insight**: Variables set by non-inline actions in a parent condition are **NOT available** to nested conditions during their evaluation because:
- Nested conditions evaluate during planning phase
- Non-inline parent actions don't execute until execution phase
- Execution phase happens AFTER all planning completes

### Solution: Use inline="true"

To make variables available to nested conditions:
```xml
<condition field="${var}" expression="^match$">
    <action application="set" data="outer_var=value" inline="true"/>  <!-- Executes during planning -->
    <condition field="${outer_var}" expression="^value$">  <!-- Now sees the variable -->
        <action application="bridge" data="destination"/>
    </condition>
</condition>
```

## Execution Order Summary

```
PLANNING PHASE (parse_exten recursively):
├─ Profile selection
├─ Context matching
└─ For each extension:
    ├─ For each condition:
    │   ├─ Evaluate condition (regex/field/time)
    │   ├─ If PASS:
    │   │   ├─ Process actions in order:
    │   │   │   ├─ inline="true" → exec_app() IMMEDIATELY
    │   │   │   └─ inline="false" → STACK in extension
    │   │   └─ If nested conditions exist:
    │   │       └─ RECURSIVE parse_exten() for children
    │   └─ Evaluate break logic
    └─ Evaluate continue="true"

EXECUTION PHASE (CS_EXECUTE state):
└─ While extension->current_application exists:
    ├─ Execute application (blocking)
    └─ Move to next application
```

## Key Implementation Constants

From `mod_dialplan_xml.c`:
```c
#define MAX_RECUR 100         // Line 82: Maximum nesting depth
#define RECUR_SPACE 4         // Line 83: Log indentation per level
```

## Variable Scope Rules

### Planning Phase Variables
**Inline actions** (`inline="true"`):
- Execute during planning
- Variables set are immediately available
- Can be used by subsequent conditions in same extension
- Available to nested conditions

**Non-inline actions** (default):
- Only stacked during planning
- Variables NOT available until execution phase
- NOT available to any conditions (parent, sibling, nested)

### Regex Captures
**Condition-scoped** (lines 507-509):
- `$1-$10`, `DP_MATCH` set per condition
- Available only within that condition's actions
- Overwritten by nested conditions
- Use `inline="true"` to save to channel variables

## Common Mistakes

### Mistake 1: Expecting non-inline variables in conditions
```xml
<condition field="${var}" expression="^match$">
    <action application="set" data="myvar=value"/>  <!-- WRONG: not inline -->
    <condition field="${myvar}" expression="^value$">  <!-- ${myvar} is empty! -->
        <action application="bridge" data="destination"/>
    </condition>
</condition>
```

**Fix**: Add `inline="true"` to the set action.

### Mistake 2: Expecting nested condition variables in parent
```xml
<condition field="${var}" expression="^match$">
    <condition field="${other}" expression="^test$">
        <action application="set" data="nested_var=123" inline="true"/>
    </condition>
    <!-- nested_var is NOT available here in parent scope -->
</condition>
```

**Reason**: Nested conditions execute after parent actions are processed. Variables flow down, not up.

### Mistake 3: Assuming execution order different from stack order
```xml
<condition field="${var}" expression="^match$">
    <action application="log" data="INFO First"/>
    <condition field="${other}" expression="^test$">
        <action application="log" data="INFO Nested"/>
    </condition>
    <action application="log" data="INFO Second"/>
</condition>
```

**Execution order**:
1. log "INFO First"
2. log "INFO Nested"
3. log "INFO Second"

Actions stack in the order encountered during depth-first traversal.

## Debug Commands

```bash
# Enable dialplan debugging
fs_cli -x "console loglevel debug"

# Enable timestamps in dialplan logs
fs_cli -x "fsctl verbose_events on"

# View dialplan execution for a call
fs_cli -x "uuid_dump <uuid>" | grep -i dialplan
```

## Performance Implications

### Planning Phase
- Fast: Only regex evaluation and list operations
- Recursive but bounded by MAX_RECUR (100 levels)
- No blocking operations (except inline actions)

### Execution Phase
- Blocking: Each application runs to completion
- Sequential: No parallelization
- Can be slow: Bridge, sleep, playback block

### Optimization Tips
1. Use `break="on-true"` to stop condition evaluation early
2. Order conditions by likelihood (most common first)
3. Minimize inline actions (they slow planning phase)
4. Use specific regex patterns (avoid greedy quantifiers)

## RFC and Standards

No direct RFC compliance - FreeSWITCH proprietary dialplan format.

## Related Documentation

- `/freeswitch/src/include/switch_caller.h` - Extension/application structures
- `/freeswitch/src/switch_core_state_machine.c` - State machine implementation
- `claude-docs/dialplan_xml_reference.md` - Application reference

## Conclusion

**The user's stated flow is CORRECT**:

```
profile → context → extension planning phase
  (condition evaluation + inline action execution + regular action stacking)
→ extension execution phase
  (executing all stacked actions)
→ continue=true check for next extension
```

**Key takeaway**: The distinction between planning and execution phases is fundamental to understanding FreeSWITCH dialplan behavior, especially regarding variable scope and action timing.

## Visual Flow Diagram

### Planning Phase - Depth-First Traversal

```
<extension name="test">
  <condition field="A" expression="match_A">          ← Line 167: Loop starts
    <action app="1" data="parent_action_1"/>          ← Line 512: Action loop
    <action app="2" data="parent_action_2"/>          ← Line 512: Next action
    <condition field="B" expression="match_B">        ← Line 582: Check nested
      <action app="3" data="nested_action_1"/>        ← Line 583: Recursive call
      <action app="4" data="nested_action_2"/>        ← (inside recursion)
    </condition>
    <action app="5" data="parent_action_3"/>          ← Line 512: Continue loop
  </condition>
</extension>

Planning order:
1. Evaluate condition A (line 409)
2. If PASS, process actions (line 512):
   - Stack or execute action 1
   - Stack or execute action 2
3. Check for nested conditions (line 582)
4. RECURSIVE call for nested condition B (line 583):
   - Evaluate condition B
   - If PASS, process nested actions:
     - Stack or execute action 3
     - Stack or execute action 4
5. Return from recursion
6. Continue parent action loop (line 512):
   - Stack or execute action 5

Stacked order for execution:
[1, 2, 3, 4, 5]
```

### Critical Code Flow with Line Numbers

```c
// mod_dialplan_xml.c:167 - Main condition loop
for (xcond = switch_xml_child(xexten, "condition"); xcond; xcond = xcond->next) {

    // Lines 409-443: Evaluate THIS condition
    proceed = switch_regex_perform(field_data, expression, &re, ovector, ...);

    if (!anti_action) {  // Line 506: Condition passed

        // Lines 512-573: Process ALL actions of THIS condition FIRST
        for (xaction = switch_xml_child(xcond, "action"); xaction; xaction = xaction->next) {
            if (xinline) {
                exec_app(session, application, app_data);  // Line 566: Execute now
            } else {
                switch_caller_extension_add_application(...); // Line 568: Stack it
            }
        }
        // All actions processed BEFORE checking nested conditions

        // Lines 581-590: NOW check for nested conditions
        if (proceed) {
            if (switch_xml_child(xcond, "condition")) {
                // Line 583: RECURSIVE call processes nested conditions
                parse_exten(session, caller_profile, xcond, extension, ...);
            }
        }
    }

    // Lines 576-578: Evaluate break logic
}
```

## Proof by Example: Complex Nesting

### Dialplan
```xml
<extension name="complex_test">
  <condition field="${level}" expression="^1$">
    <action application="log" data="INFO Action A1"/>
    <action application="log" data="INFO Action A2"/>

    <condition field="${level}" expression="^2$">
      <action application="log" data="INFO Action B1"/>

      <condition field="${level}" expression="^3$">
        <action application="log" data="INFO Action C1"/>
      </condition>

      <action application="log" data="INFO Action B2"/>
    </condition>

    <action application="log" data="INFO Action A3"/>
  </condition>
</extension>
```

### Planning Phase Walkthrough

```
parse_exten(extension="complex_test", recur=0)
├─ Evaluate: ${level} =~ /^1$/ → PASS
├─ Process parent actions (lines 512-573):
│  ├─ STACK: log("INFO Action A1")         [Queue position: 1]
│  └─ STACK: log("INFO Action A2")         [Queue position: 2]
├─ Check nested conditions (line 582) → Found
└─ RECURSIVE: parse_exten(recur=1)
   ├─ Evaluate: ${level} =~ /^2$/ → PASS
   ├─ Process actions:
   │  └─ STACK: log("INFO Action B1")      [Queue position: 3]
   ├─ Check nested conditions → Found
   └─ RECURSIVE: parse_exten(recur=2)
      ├─ Evaluate: ${level} =~ /^3$/ → PASS
      ├─ Process actions:
      │  └─ STACK: log("INFO Action C1")   [Queue position: 4]
      └─ Check nested conditions → None
   ├─ Continue parent (level 2) actions:
   │  └─ STACK: log("INFO Action B2")      [Queue position: 5]
   └─ Return from recur=1
├─ Continue parent (level 1) actions:
│  └─ STACK: log("INFO Action A3")         [Queue position: 6]
└─ Return from recur=0
```

### Execution Phase Output

```
[DEBUG] Dialplan: Action log(INFO Action A1)
INFO Action A1

[DEBUG] Dialplan: Action log(INFO Action A2)
INFO Action A2

[DEBUG] Dialplan: Action log(INFO Action B1)
INFO Action B1

[DEBUG] Dialplan: Action log(INFO Action C1)
INFO Action C1

[DEBUG] Dialplan: Action log(INFO Action B2)
INFO Action B2

[DEBUG] Dialplan: Action log(INFO Action A3)
INFO Action A3
```

**Order**: A1, A2, B1, C1, B2, A3

This is **depth-first, pre-order traversal** where:
- Parent actions BEFORE nested conditions are processed first
- Then nested condition actions
- Then parent actions AFTER nested conditions

## Source Code Evidence Table

| Behavior | Code Location | Evidence |
|----------|---------------|----------|
| Conditions evaluated in planning | `mod_dialplan_xml.c:409` | `switch_regex_perform()` called during hunt |
| Inline actions execute immediately | `mod_dialplan_xml.c:566` | `exec_app()` called when `xinline = true` |
| Non-inline actions stacked | `mod_dialplan_xml.c:568` | `switch_caller_extension_add_application()` |
| Parent actions before nested eval | `mod_dialplan_xml.c:512-573` then `582-589` | Action loop completes before nested check |
| Nested conditions during planning | `mod_dialplan_xml.c:583` | Recursive `parse_exten()` call |
| Execution phase separate | `switch_core_state_machine.c` | `CS_EXECUTE` state loops through applications |
| Max nesting depth | `mod_dialplan_xml.c:82` | `#define MAX_RECUR 100` |

## Edge Cases and Special Behaviors

### Edge Case 1: Actions After Nested Conditions
```xml
<condition field="${var}" expression="match">
  <action application="log" data="INFO Before"/>
  <condition field="${other}" expression="test">
    <action application="log" data="INFO Nested"/>
  </condition>
  <action application="log" data="INFO After"/>  <!-- This STACKS after nested actions -->
</condition>
```

**Execution order**: Before → Nested → After

**Why**: The XML parser processes child nodes in document order. When `parse_exten()` encounters the nested `<condition>` (line 582), it recursively processes it, THEN continues the parent's action loop at line 512 for remaining actions.

### Edge Case 2: Break in Nested Condition
```xml
<condition field="${var}" expression="match">
  <action application="log" data="INFO Parent1"/>
  <condition field="${other}" expression="test" break="on-true">
    <action application="log" data="INFO Nested"/>
  </condition>
  <action application="log" data="INFO Parent2"/>  <!-- Still executes -->
</condition>
```

**Execution order**: Parent1 → Nested → Parent2

**Why**: The `break` attribute (line 576) only affects the condition loop within that recursion level. It doesn't prevent the parent from continuing.

### Edge Case 3: Multiple Nested Conditions
```xml
<condition field="${var}" expression="match">
  <action application="log" data="INFO A1"/>
  <condition field="${b}" expression="test">
    <action application="log" data="INFO B"/>
  </condition>
  <action application="log" data="INFO A2"/>
  <condition field="${c}" expression="test">
    <action application="log" data="INFO C"/>
  </condition>
  <action application="log" data="INFO A3"/>
</condition>
```

**Execution order**: A1 → B → A2 → C → A3

**Why**: Each nested condition is processed as it's encountered during the action loop traversal.

### Edge Case 4: Inline Actions in Nested Conditions
```xml
<condition field="${var}" expression="match">
  <action application="set" data="parent=1"/>
  <condition field="${parent}" expression="^1$">  <!-- Sees empty string -->
    <action application="bridge" data="dest"/>
  </condition>
</condition>

<!-- Fix: -->
<condition field="${var}" expression="match">
  <action application="set" data="parent=1" inline="true"/>  <!-- Executes immediately -->
  <condition field="${parent}" expression="^1$">  <!-- NOW sees "1" -->
    <action application="bridge" data="dest"/>
  </condition>
</condition>
```

**Critical**: Inline actions execute during planning (line 566), so they modify channel state before nested conditions evaluate.

## State Machine Context

From `switch_core_state_machine.c`:

```
Channel State Flow:
CS_NEW
  → CS_INIT
    → CS_ROUTING (dialplan_hunt() called here)
      → CS_EXECUTE (actions execute here)
        → CS_HANGUP
```

### CS_ROUTING State (Planning Phase)
- Calls `switch_core_session_run_indication()` → dialplan interface
- Dialplan interface calls `dialplan_hunt()` (line 620)
- `dialplan_hunt()` returns `switch_caller_extension_t*` with stacked actions
- Extension stored via `switch_channel_set_caller_extension()`
- State machine transitions to CS_EXECUTE

### CS_EXECUTE State (Execution Phase)
- Retrieves extension via `switch_channel_get_caller_extension()`
- Loops through `extension->current_application` linked list
- Each application executed via `switch_core_session_execute_application()`
- Applications block until completion
- When list exhausted, moves to next state (HANGUP or RESET)

## Memory and Performance Details

### Action Stack Structure
From `switch_caller.h`:

```c
struct switch_caller_application {
    char *application_name;
    char *application_data;
    struct switch_caller_application *next;
};

struct switch_caller_extension {
    const char *extension_name;
    const char *extension_number;
    switch_caller_application_t *applications;
    switch_caller_application_t *current_application;
    switch_caller_application_t *last_application;
};
```

Actions stored as singly-linked list, appended during planning phase.

### Memory Allocation
**Line 530-537**: Regex substitution buffer allocation
```c
len = (strlen(data) + strlen(field_data) + 10) * proceed;
substituted = malloc(len);
```

**Implication**: Complex regex with many captures can consume significant memory during planning.

### Planning Phase Performance
- **Fast operations**: Regex evaluation, string comparison, list append
- **Slow operations**: Inline applications (blocking)
- **Recursion overhead**: Minimal, bounded by MAX_RECUR (100)

### Execution Phase Performance
- **Fully sequential**: No parallelization
- **Blocking**: Each app must complete before next starts
- **State checks**: Channel state checked between apps (line while condition)

## Comparison with Other Dialplan Systems

### Asterisk extensions.conf
- **Planning**: Pattern matching only
- **Execution**: Immediate per priority
- **No separation**: Each priority executes as encountered

### FreeSWITCH Advantages
- Full plan compiled before execution
- Can inspect/modify plan programmatically
- Inline actions allow conditional stacking

### FreeSWITCH Disadvantages
- Two-phase model confusing for beginners
- Variable scope issues with non-inline actions
- No way to modify stacked actions after planning

## Advanced Techniques

### Technique 1: Conditional Action Stacking
Use inline conditions to control whether actions are stacked:

```xml
<condition field="${route_type}" expression="^direct$">
  <action application="set" data="use_direct=true" inline="true"/>
</condition>

<condition field="${use_direct}" expression="^true$">
  <action application="bridge" data="direct_gateway/${destination}"/>
  <anti-action application="bridge" data="failover_gateway/${destination}"/>
</condition>
```

### Technique 2: Multi-Stage Evaluation
Use nested conditions for progressive refinement:

```xml
<condition field="${caller_id_number}" expression="^(\d{10})$">
  <action application="set" data="normalized_cli=$1" inline="true"/>

  <condition field="${db(select name from users where cli='${normalized_cli}')}"
             expression="^(.+)$">
    <action application="set" data="user_name=$1" inline="true"/>

    <condition field="${user_name}" expression="^premium_">
      <action application="set" data="codec_preference=OPUS"/>
      <anti-action application="set" data="codec_preference=PCMU"/>
    </condition>
  </condition>
</condition>
```

### Technique 3: Action Ordering Control
Place critical inline actions before conditional branches:

```xml
<condition field="${destination_number}" expression="^(.+)$">
  <action application="set" data="call_start_time=${strftime(%s)}" inline="true"/>
  <action application="db" data="insert/calls/${uuid}/${call_start_time}" inline="true"/>

  <!-- Now safe to branch - tracking already done -->
  <condition field="${account_balance}" expression="^[1-9]\d*$">
    <action application="bridge" data="${destination}"/>
  </condition>
</condition>
```

