---
description: Release a ticpu-cc plugin (version bump, commit, signed tag, push)
argument-hint: <plugin> <version>
allowed-tools: Bash, Read, Edit
---

Release the `$1` plugin at version `$2`.

This is a multi-plugin marketplace, so each plugin versions and tags
independently. `bcachefs` is a submodule and is released in its own repo, not
here — only plugins with a `<plugin>/.claude-plugin/plugin.json` in this repo
are released by this command (currently `freeswitch`).

## Steps

1. **Pre-flight**
   - `git status` — confirm the working tree holds only the changes meant for
     this release. Stage explicit paths only; never `git add -A/./-u`.
   - Read `$1/.claude-plugin/plugin.json` and confirm the current `version`.
   - Confirm `$2` is a sane bump from it (patch/minor/major) and follows semver.

2. **Commit content first, release bump last.** Each logical change is its own
   commit (conventional commits: `docs(...)`, `feat(...)`, `fix(...)`). Do NOT
   fold content changes into the release commit. Unrelated working-tree changes
   get committed separately on their own topic before the release.

3. **Bump version** in `$1/.claude-plugin/plugin.json` to `$2`.

4. **Release commit** — stage only `$1/.claude-plugin/plugin.json`:

   ```
   git commit -m "release: $1 v$2"
   ```

5. **Signed annotated tag**, plugin-prefixed, changelog in the tag message
   (features/fixes/API changes for someone not following dev — not commit
   hashes):

   ```
   git tag -as $1-v$2 -F - <<'EOF'
   $1 v$2

   - <changelog line>
   - <changelog line>
   EOF
   ```

6. **Push** commits and the tag:

   ```
   git push && git push origin $1-v$2
   ```

7. **Report** the update commands for consumers:

   ```
   claude plugin marketplace update ticpu-cc
   claude plugin update $1@ticpu-cc
   ```

## Rules

- Commit trailer: only `Co-Authored-By:` — model from the "You are powered by"
  line in the environment. No emoji / generated-by lines.
- Tag name is `<plugin>-v<version>` (e.g. `freeswitch-v0.3.0`) to disambiguate
  plugins sharing this repo. Bare `v<version>` is reserved for a single-plugin
  layout and is not used here.
- Never release without the explicit instruction to do so (this command is that
  instruction). Bumping version / tagging / pushing tags all happen only here.
