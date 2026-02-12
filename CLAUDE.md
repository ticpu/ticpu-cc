# ticpu-cc

Personal Claude Code skill marketplace by Jérôme Poulin.

## Plugins

- `freeswitch` — skills: `usage` (variables, ESL, user guide), `dialplan` (dialplan architect, dptools, channel variables), `dev` (core C, codecs, CI/CD), `sofia` (mod_sofia SIP)
- `bcachefs` — submodule from `ticpu/bcachefs-claude-plugin`

## FreeSWITCH

- Source code is at /home/jerome.poulin/GIT/freeswitch/freeswitch
- Always use the source code to ground your facts.
- Do not use *critical* or add markers that suggest some topics are more important than others. Prefer modifying existing documentation or reformulating. Do not fall for the "additive solution bias", review the documentation properly and prefer adjusting existing ones unless it is actually something new to learn.

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
