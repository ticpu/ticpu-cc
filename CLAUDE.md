# ticpu-cc

Personal Claude Code skill marketplace by Jérôme Poulin.

## Skills

- `freeswitch` — config/usage: dialplan, variables, ESL, time routing, basic SIP
- `freeswitch-dev` — internals/build: core media C code, codec implementations, CI/CD
- `sofia` — mod_sofia SIP expert: flags, constants, NUA, RTP, NAT, auth, timers
- `bcachefs` — synced from `vendor/bcachefs-claude-plugin` (submodule → `ticpu/bcachefs-claude-plugin`)

## Syncing bcachefs skill

```
cd vendor/bcachefs-claude-plugin && git pull origin master
```

## Releasing

1. Bump `version` in `.claude-plugin/plugin.json`
2. Commit: `release: v<version>`
3. Annotated signed tag: `git tag -as v<version>` with changelog body
4. Push: `git push && git push --tags`

## Updating after push

```
claude plugin marketplace update ticpu-cc
claude plugin update freeswitch@ticpu-cc
```
