# ticpu-cc

Personal Claude Code skill marketplace by Jérôme Poulin.

## Plugins

- `freeswitch` — skills: `usage` (dialplan, variables, ESL), `dev` (core C, codecs, CI/CD), `sofia` (mod_sofia SIP)
- `bcachefs` — submodule from `ticpu/bcachefs-claude-plugin`

## Syncing bcachefs

```
cd vendor/bcachefs-claude-plugin && git pull origin master
```

## Releasing

1. Bump `version` in `<plugin>/.claude-plugin/plugin.json`
2. Commit: `release: v<version>`
3. Annotated signed tag: `git tag -as v<version>` with changelog body
4. Push: `git push && git push --tags`

## Updating after push

```
claude plugin marketplace update ticpu-cc
claude plugin update <name>@ticpu-cc
```
