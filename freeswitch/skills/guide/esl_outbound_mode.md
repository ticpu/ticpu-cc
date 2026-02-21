# ESL Outbound Mode

Outbound mode: FreeSWITCH connects to an external application via the
`socket` dialplan application (mod_event_socket). The application listens
on a TCP port and accepts connections from FreeSWITCH.

## Socket Application

Registered by mod_event_socket, used in dialplan like any dptools app:

```xml
<action application="socket" data="127.0.0.1:8040 async full"/>
```

Parameters: `<host>:<port> [async [full]]`

- Without `async`: channel blocks until each command completes
- Without `full`: only single-channel commands available
- With `async full`: full ESL command set

## Protocol Sequence

1. FreeSWITCH connects to the listening app
2. App sends `connect\n\n` — mandatory first command
3. FreeSWITCH responds with channel data (all channel variables as headers)
4. App sends setup commands: `myevents`, `linger`, `resume`, `filter`
5. App sends execution commands: `sendmsg` (execute apps), `api`, `bgapi`

## Command Availability by Mode

| Command | single-channel | async full |
|---|---|---|
| connect | yes | yes |
| myevents | yes | yes |
| getvar | yes | yes |
| resume | yes | yes |
| filter | yes | yes |
| divert_events | yes | yes |
| sendmsg | yes | yes |
| linger / nolinger | no | yes |
| event / nixevent / noevents | no | yes |
| api / bgapi | no | yes |
| sendevent | no | yes |
| log / nolog | no | yes |

Source: `mod_event_socket.c` — `LFLAG_FULL` guard skips commands after
`sendmsg` for outbound connections without the `full` flag.

## Key Commands

### connect

Mandatory first command. Returns all channel variables as response headers.
Response includes mode confirmation:

- `Control: full` vs `Control: single-channel`
- `Socket-Mode: async` vs `Socket-Mode: static`

### linger

```
linger\n\n
linger 600\n\n
```

Keeps the socket open after the channel hangs up. Without linger, the
socket closes immediately on hangup. With linger, FreeSWITCH sends a
`text/disconnect-notice` with `Content-Disposition: linger` and keeps the
socket open so the client can drain remaining events.

- Bare `linger` = indefinite (original form, always supported)
- `linger <seconds>` = timed linger (auto-close after timeout)
- Requires `async full` mode

### nolinger

Cancels linger mode.

### resume

```
resume\n\n
```

Tells FreeSWITCH to continue dialplan execution from the next action
after the `socket` application when the ESL connection disconnects.

- Without resume: channel is hung up when socket disconnects
- With resume: dialplan continues from where it left off (next action
  after `<action application="socket" .../>`)
- Available in both single-channel and full mode

### myevents

```
myevents plain\n\n
myevents json <uuid>\n\n
```

Subscribe to events for this channel only. Format: plain, json, or xml.

## Originate with Socket App

The socket app data contains spaces (`host:port async full`), but
originate's parser splits on spaces. Single-quote the application:

```
originate loopback/ext/ctx '&socket(127.0.0.1:8040 async full)'
```

## Typical Outbound Setup

```xml
<!-- Dialplan -->
<action application="socket" data="127.0.0.1:8040 async full"/>
<action application="hangup"/>  <!-- runs if resume enabled and app disconnects -->
```

```
→ connect                    (get channel data)
→ myevents plain             (subscribe to channel events)
→ linger                     (keep socket open after hangup)
→ resume                     (continue dialplan on disconnect)
→ sendmsg execute answer     (answer the call)
→ sendmsg execute playback   (play audio)
← events...                  (receive channel events)
```
