---
name: guide
description: >
  ALWAYS load this skill before answering any FreeSWITCH configuration
  or operational question. Covers channel variable scoping and lifecycle,
  hangup causes, ESL authentication and userauth (per-user permissions,
  ACL, command filtering), ESL JSON API, time-of-day routing with PCRE
  regex, SIP profile configuration and debugging, state machine, and
  general operational guidance. Also load the dialplan skill for
  writing/reviewing dialplans, the sofia skill for SIP-specific questions,
  or the dev skill for C code and codec internals.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Task
---

# FreeSWITCH Assistant

You are helping with FreeSWITCH configuration and usage. FreeSWITCH is a
modular telephony platform. This skill covers configuration, variables, ESL,
state machine, and basic SIP operation.

## Key Concepts

- **Hunt-then-execute model**: Dialplan has two phases — conditions are
  evaluated during hunt, actions run during execute. Variables set in execute
  are unavailable to hunt-phase conditions.
- **Inline execution**: Only `SAF_ROUTING_EXEC` applications (set, export,
  hash, etc.) work during hunt phase via `inline="true"`.
- **Channel variables**: Key-value store per call leg, set/read throughout
  the call lifecycle with scoping rules (local vs inherited vs exported).
- **Event system**: Pub/sub with subclasses; `switch_event_t` carries
  headers + body.
- **State machine**: CS_NEW → CS_INIT → CS_ROUTING → CS_EXECUTE →
  CS_EXCHANGE_MEDIA → CS_HANGUP → CS_REPORTING → CS_DESTROY.
- **SIP profiles**: Each `sofia_profile` binds to an IP:port and handles
  SIP signaling. Profiles have gateways for outbound registration. Basic
  debugging: `sofia status`, `sofia status profile <name>`,
  `sofia loglevel all 9`, `sofia global siptrace on`.
- **ESL auth modes**: `auth <password>` (global, full access) vs
  `userauth user@domain:password` (per-user, with event/API/log restrictions
  from XML directory). ACL checked before auth challenge.
- **ESL userauth parameters**: `esl-password`, `esl-allowed-events`,
  `esl-allowed-api`, `esl-allowed-log`, `esl-disable-command-logging` —
  set in directory `<params>` at domain/group/user level (later overrides).

## Hard Limits

```c
#define MAX_RECUR 100          // Dialplan nesting depth
int ovector[30]                // PCRE capture limit (15 pairs)
#define MAX_ACL 100            // Event socket ACL entries
#define MAX_CMD_LOG_EXCLUDE 32 // Command log exclusion patterns
```

## Detailed Reference

Read the reference docs bundled with this skill (same directory as this file).
Pick the relevant ones based on $ARGUMENTS before answering questions.

- [dialplan_contexts.md](dialplan_contexts.md) - Context structure, extension matching, continue flag
- [dialplan_execution_flow.md](dialplan_execution_flow.md) - Hunt vs execute phases
- [dialplan_execution_flow_detailed.md](dialplan_execution_flow_detailed.md) - Detailed execution flow with code paths
- [dialplan_xml_reference.md](dialplan_xml_reference.md) - XML dialplan syntax reference
- [nested_conditions_behavior.md](nested_conditions_behavior.md) - Nested condition evaluation logic
- [freeswitch_time_conditions_and_regex.md](freeswitch_time_conditions_and_regex.md) - Time-of-day routing, PCRE regex
- [freeswitch_variables_guide.md](freeswitch_variables_guide.md) - Variable scoping, lifecycle, expansion
- [esl_authentication.md](esl_authentication.md) - ESL auth, userauth, per-user permissions, ACL, command logging
- [esl_api_json_formatting.md](esl_api_json_formatting.md) - ESL JSON API formatting
- [esl_outbound_mode.md](esl_outbound_mode.md) - ESL outbound mode: socket app, connect/linger/resume, command availability
- [esl_event_reference.md](esl_event_reference.md) - Event headers, Caller- prefix convention, CUSTOM subclasses, priority, hangup causes
- [sip_freeswitch_master_guide.md](sip_freeswitch_master_guide.md) - SIP debugging overview and common patterns

## Guidelines

- Consult the reference docs first, verify with source code when needed
- Back answers with file:line evidence
- Trust source code over community wiki (may be outdated)
- For writing/reviewing dialplans, dptools, channel variables → use the `dialplan` skill
- For deep mod_sofia internals (flags, constants, NUA) → use the `sofia` skill
- For core C code, codec implementations, build system → use the `dev` skill
