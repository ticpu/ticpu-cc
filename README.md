# ticpu-cc

Personal Claude Code skill marketplace.

## Install

```
claude plugin marketplace add ticpu/ticpu-cc
claude plugin install freeswitch@ticpu-cc
```

## Browse available skills

```
/plugin marketplace list
/plugin
```

The `/plugin` interactive UI shows a **Discover** tab to browse all skills
in configured marketplaces.

## Usage

```
/freeswitch          Dialplan, variables, ESL, time routing, basic SIP
/freeswitch-dev      Core media C code, codec implementations, CI/CD
/sofia               mod_sofia SIP internals, flags, constants, RTP, NAT
```

## Update

```
claude plugin marketplace update ticpu-cc
claude plugin update freeswitch@ticpu-cc
```

## Skills

### freeswitch

FreeSWITCH configuration and usage: XML dialplan execution (hunt vs execute
phases, inline execution, nested conditions), channel variable scoping,
ESL JSON API, time-of-day routing, basic SIP profile debugging.

### freeswitch-dev

FreeSWITCH core C development: core media framework (switch_core_media),
codec negotiation internals (Normal/Greedy/Scrooge), Opus/AMR-WB codec
implementations, transcoding bridge IO path, Oreka recording, Cauca CI/CD.

### sofia

FreeSWITCH mod_sofia SIP expert: mod_sofia internals, libsofia-sip NUA
integration, 765+ constants, 120+ flags, SIP timers, RTP/SRTP, NAT handling,
authentication (RFC 8760), gateway state machines, transport management.

## License

MPL-1.1 (documentation derived from FreeSWITCH source, licensed under MPL 1.1)
