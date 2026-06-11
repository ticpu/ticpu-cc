# Pre-Dialplan (mod_sofia)

CAUCA-specific mod_sofia feature (branch `v1.10.13/mod_sofia_pre_dialplan`,
`sofia_pre_dialplan.c`). Not in upstream FreeSWITCH.

## Purpose

Header-based SIP processing inside mod_sofia, before the call reaches the XML
dialplan. Handles common cases (early rejection, header inspection, flag
setting) that would otherwise need an external SIP proxy like Kamailio.

## Hooks

Extensions run at one of two points, selected by the `hook` attribute:

- `pre-auth` — earliest point, before authentication. Only a minimal set of
  channel variables exists (see below). Default when an extension wants to set
  tflags that influence later parsing/auth.
- `post-parsing` — after most headers are parsed; full channel-variable set is
  available. This is the **default** when `hook` is omitted or empty.
- `disabled` — extension is parsed but never executed.

## Variable availability by hook

`pre-auth` exposes only a minimal set:

- `direction`, `uuid`, `call_uuid`, `session_id`
- `sip_from_user`, `sip_from_uri`, `sip_from_host`
- `sip_referred_by_uri`, `sip_referred_by_host`
- `channel_name`, `sip_call_id`

`post-parsing` exposes essentially the full set normally seen in the dialplan.
Setting the `PARSE_ALL_INVITE_HEADERS` tflag in `pre-auth` makes the unedited
headers available as `sip_i_*` variables in `post-parsing`.

## Configuration

Declared per sofia profile via a `pre-dialplan` param:

```xml
<param name="pre-dialplan">
  <extension name="..." match="all|any" continue="true|false" hook="pre-auth|post-parsing|disabled">
    <condition channel_variable="..." expression="..."/>
    <action type="..." data="..."/>
  </extension>
</param>
```

- `match` — `all` (default) requires every condition to match; `any` matches on
  the first hit.
- `continue` — `true` keeps evaluating later extensions after this one matches;
  `false` (default) stops.

## Actions

| Action | Syntax | Effect |
|--------|--------|--------|
| `set_tflag` | `data="FLAG_NAME"` | Set a technical flag (see below) |
| `log` | `data="LEVEL message"` | Write a log message |
| `info` | `data="LEVEL"` | Dump all available channel variables at log LEVEL |
| `set` | `data="var=value"` | Set a channel variable |
| `respond` | `data="CODE reason"` | Send a SIP response (`nua_respond`) |
| `terminate` | — | Stop processing the call |

## Technical flags (`set_tflag`)

| Flag | Effect |
|------|--------|
| `PARSE_ALL_INVITE_HEADERS` | Force parsing of all SIP headers, exposing `sip_i_*` in post-parsing |
| `3PCC_FORCE_DIALPLAN` | Route 3PCC calls through the dialplan before answering 200 OK |
| `FORCE_AUTHENTICATED` | Skip authentication |
| `LATE_NEGOTIATION` | Enable late media negotiation for this call |

## Header-parsing pattern

Enable parsing in `pre-auth`, consume the parsed headers in `post-parsing`:

```xml
<extension name="enable-parsing" hook="pre-auth" continue="true">
  <action type="set_tflag" data="PARSE_ALL_INVITE_HEADERS"/>
</extension>

<extension name="use-headers" hook="post-parsing">
  <condition channel_variable="sip_i_custom_header" expression=".*"/>
  <action type="set" data="found_custom=true"/>
</extension>
```

Parse selectively to save CPU — only set the tflag when a cheap condition says
the headers are needed.

## Replacing Kamailio patterns

Early rejection of a scanner, equivalent to a Kamailio `sl_send_reply` + `exit`:

```xml
<extension name="check-ua" hook="post-parsing">
  <condition channel_variable="sip_user_agent" expression=".*scanner.*"/>
  <action type="respond" data="403 Forbidden"/>
  <action type="terminate"/>
</extension>
```
