# ESL Authentication and Authorization

mod_event_socket supports two authentication modes for inbound connections: global password auth and per-user directory auth (userauth). Both are enforced in `parse_command()` before any other commands are accepted.

Source: `src/mod/event_handlers/mod_event_socket/mod_event_socket.c`

## Connection Flow

1. Client connects to TCP port (default 8021)
2. ACL check against `apply-inbound-acl` (default: `loopback.auto`) — rejected clients get "Access Denied, go away." (line 2737-2748)
3. Server sends `Content-Type: auth/request\n\n`
4. Client sends `auth <password>` or `userauth <user>@<domain>:<password>`
5. Only `exit`, `auth`, and `userauth` are accepted before `LFLAG_AUTHED` is set (line 1807)
6. Failed auth sets `-ERR invalid` and clears `LFLAG_RUNNING` (disconnects)

## Password Auth (`auth`)

Simple comparison against `prefs.password` from `event_socket.conf.xml` (line 1814).

```
auth ClueCon
```

No per-connection restrictions — full access to all events, APIs, and logs.

## User Auth (`userauth`)

Per-user authentication against the FreeSWITCH XML directory. Two wire formats are accepted (lines 1847-1858):

```
userauth user@domain:password
userauth user:password@domain
```

Parsing logic: first tries `user@domain:password` split (@ then :), then falls back to `user:password@domain` (: then @). Both user and domain are required (line 1860).

### Directory Lookup

The lookup uses `switch_xml_locate_user()` with action `event_socket_auth` (line 1877-1879):

```c
switch_event_add_header_string(params, SWITCH_STACK_BOTTOM, "action", "event_socket_auth");
switch_xml_locate_user("id", user, domain_name, NULL, &x_domain_root, &x_domain, &x_user, &x_group, params);
```

This means mod_xml_curl and other directory providers can serve ESL users dynamically by checking `action=event_socket_auth` in the request.

### Parameter Hierarchy

Parameters are read from three levels, iterated in order — later values override earlier (lines 1883-1906):

1. `<domain>` → `<params>`
2. `<group>` → `<params>`
3. `<user>` → `<params>`

### Directory Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `esl-password` | string | (required) | Must match the password in the userauth command |
| `esl-allowed-events` | string | all | Comma or space-separated event types the user can subscribe to |
| `esl-allowed-api` | string | all | Comma or space-separated API commands the user can execute |
| `esl-allowed-log` | boolean | `true` | Whether the user can receive log output |
| `esl-disable-command-logging` | boolean | `false` | Suppress command logging for this user |

Delimiter detection: if the value contains a space, space is used; otherwise comma (lines 1930, 1984).

### Success Response

On successful userauth, the reply includes the granted permissions (line 2009):

```
Content-Type: command/reply
Reply-Text: +OK accepted
Allowed-Events: all
Allowed-API: show sofia status version uptime
Allowed-LOG: true
```

## Authorization Enforcement

### API Command Authorization

When `allowed_api_hash` is set, every `api` and `bgapi` command is checked via `auth_api_command()` (lines 1641-1694, 2359-2378, 2413-2418).

Only the command name (first word) is checked — subcommands cannot be filtered individually.

**Sneaky command detection** (line 1644): the function recursively checks through wrapper commands that could bypass restrictions:

```c
char *sneaky_commands[] = { "bgapi", "sched_api", "eval", "expand", "xml_wrap", NULL };
```

Example: if a user is allowed `show` but not `sofia`, the command `expand sofia status` is rejected because `auth_api_command()` unwraps `expand` and checks the inner `sofia` command.

### Event Subscription Authorization

When `LFLAG_AUTH_EVENTS` is set, the `event` command checks each requested event type against `allowed_event_list[]` (line 2566-2570):

```c
if (switch_test_flag(listener, LFLAG_AUTH_EVENTS) && !listener->allowed_event_list[type] &&
    !switch_test_flag(listener, LFLAG_ALL_EVENTS_AUTHED)) {
    switch_snprintf(reply, reply_len, "-ERR permission denied");
```

Custom event subclasses are checked against `allowed_event_hash` (line 2559).

When `esl-allowed-events` contains `ALL`, the flag `LFLAG_ALL_EVENTS_AUTHED` is set and all events are permitted (line 1946).

### Log Access

The `log` command checks `LFLAG_ALLOW_LOG` and returns `-ERR permission denied` if not set (line 2464-2467). This flag is cleared at the start of userauth processing and only re-set if `esl-allowed-log` is true (lines 1841, 1968-1970).

## Server Configuration

`autoload_configs/event_socket.conf.xml`:

```xml
<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <param name="listen-ip" value="127.0.0.1"/>
    <param name="listen-port" value="8021"/>
    <param name="password" value="ClueCon"/>
    <!-- ACL applied to all inbound connections (default: loopback.auto) -->
    <param name="apply-inbound-acl" value="lan"/>
    <!-- Command logging level (default: INFO, use "none" to disable) -->
    <param name="command-log-level" value="INFO"/>
    <!-- PCRE regex patterns to exclude from command logging -->
    <param name="command-log-exclude" value="^api status$"/>
    <param name="nat-map" value="false"/>
    <param name="stop-on-bind-error" value="true"/>
  </settings>
</configuration>
```

### ACL Behavior

- Up to 100 ACL entries (`MAX_ACL`, line 123)
- If no `apply-inbound-acl` is configured, defaults to `loopback.auto` (line 3028-3029)
- ACL is checked immediately after TCP accept, before auth challenge (line 2737)
- Rejected connections are closed with no auth/request sent

### Command Logging

- `command-log-level`: log level for ESL commands (default INFO, `none`/`off`/`disabled` to suppress)
- `command-log-exclude`: PCRE regex patterns (up to 32, validated at config load)
- `auth` and `userauth` commands are always excluded from logging (hardcoded, line 1737)
- Per-user: `esl-disable-command-logging` suppresses logging for that user (line 1732)
- Log output: `Event Socket Command from <ip>:<port>: <command>` (line 1772-1774)
- Commands truncated at `MAX_LOG_CMD_LEN` with ellipsis

## Directory User Examples

### Full-Access Admin

```xml
<user id="admin">
  <params>
    <param name="esl-password" value="SuperSecretPassword"/>
  </params>
</user>
```

### Restricted Monitoring User

```xml
<user id="monitor">
  <params>
    <param name="esl-password" value="MonitorPass"/>
    <param name="esl-allowed-events" value="CHANNEL_CREATE CHANNEL_ANSWER CHANNEL_HANGUP CHANNEL_HANGUP_COMPLETE"/>
    <param name="esl-allowed-api" value="show status version uptime"/>
    <param name="esl-allowed-log" value="false"/>
  </params>
</user>
```

### Event-Only User (No API)

```xml
<user id="cdr-collector">
  <params>
    <param name="esl-password" value="CdrOnly"/>
    <param name="esl-allowed-events" value="CHANNEL_HANGUP_COMPLETE CUSTOM"/>
    <param name="esl-allowed-api" value=""/>
    <param name="esl-allowed-log" value="false"/>
  </params>
</user>
```

### Domain-Wide Defaults with Overrides

```xml
<domain name="default">
  <params>
    <param name="esl-allowed-log" value="false"/>
    <param name="esl-allowed-api" value="show status version"/>
  </params>
  <groups>
    <group name="admins">
      <params>
        <param name="esl-allowed-log" value="true"/>
      </params>
      <users>
        <user id="superadmin">
          <params>
            <param name="esl-password" value="SuperAdmin123"/>
            <param name="esl-allowed-api" value="all"/>
          </params>
        </user>
      </users>
    </group>
    <group name="operators">
      <users>
        <user id="operator1">
          <params>
            <param name="esl-password" value="Operator1Pass"/>
          </params>
        </user>
      </users>
    </group>
  </groups>
</domain>
```

## Listener Flags Reference

| Flag | Bit | Purpose |
|---|---|---|
| `LFLAG_AUTHED` | 0 | Set after successful auth/userauth |
| `LFLAG_AUTH_EVENTS` | 14 | Event subscriptions restricted to `allowed_event_list[]` |
| `LFLAG_ALL_EVENTS_AUTHED` | 15 | All standard events permitted (CUSTOM still filtered by hash) |
| `LFLAG_ALLOW_LOG` | 16 | `log` command permitted |

## Listener Struct Fields

| Field | Type | Purpose |
|---|---|---|
| `allowed_event_list[]` | `uint8_t[SWITCH_EVENT_ALL+1]` | Per-event-type allow bitmap |
| `allowed_event_hash` | `switch_hash_t*` | Allowed CUSTOM event subclass names |
| `allowed_api_hash` | `switch_hash_t*` | Allowed API command names |
| `disable_cmd_logging` | `uint8_t` | Per-user command log suppression |
