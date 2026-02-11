# FreeSWITCH CAUCA Build

## CI Pipeline (`freeswitch-ci.sh`)

### Commands
- `./freeswitch-ci.sh compile` - Build FreeSWITCH (internal deps + full build)
- `./freeswitch-ci.sh compile -f` - Fast build (FreeSWITCH only, skip deps)
- `./freeswitch-ci.sh build-builder` - Build builder image with external deps
- `./freeswitch-ci.sh build-builder --base-only` - Build base image only

### Container Architecture
```
builder-base    → Debian + apt packages + luarocks
builder-latest  → builder-base + external deps in /usr
compile         → builder-latest + sofia-sip + FreeSWITCH
```

### Dependencies (`docker/cauca/install-deps.sh`)
- **External** (baked into builder image): ogg, vorbis, flac, opus, ffmpeg, libsndfile, spandsp, etc.
- **Internal** (built at compile time): sofia-sip from `cauca-ng911` branch
- Format: `name|url|flags|branch` - branch field is optional
- Install prefix: `/usr` (not /usr/local)

### Key Files
- `freeswitch-ci.sh` - Main CI script with `ctr_exec()` helper
- `docker/cauca/install-deps.sh` - Dependency builder (verb commands: build-external, build-internal, build-all)
- `docker/cauca/Containerfile.base` - Base image (Debian trixie + tools)
- `docker/cauca/Containerfile` - Builder image with external deps
- `.gitlab-ci.yml` - Pipeline config

### Builder Image Tagging
- Format: `builder-YYYYWWW-<hash>` (e.g., `builder-2025W48-414997c6`)
- Hash from: Containerfile.base, Containerfile, install-deps.sh, luarocks-config.lua
- Skip rebuild if hash unchanged (unless tag build or web trigger)

### CI Environment
- `CI` variable triggers SSH→HTTPS URL conversion for GitHub
- `CI_JOB_TOKEN` used for internal GitLab repos (passed via `-e` to container exec)
- Host SSL certs mounted: `-v /etc/ssl/certs:/etc/ssl/certs:ro`
- BuildKit enabled with cache mounts for apt/luarocks

### Gotchas
- Container exec inherits ENV from Dockerfile, but verify with `podman exec ... env`
- Autotools timestamp issues after git clone: use `fix_autotools_timestamps()`
- Docker runs as root - files created in mounted volumes are root-owned
- "no usable sofia-sip" usually means version mismatch, not missing
