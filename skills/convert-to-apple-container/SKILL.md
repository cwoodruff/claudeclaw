---
name: convert-to-apple-container
description: Switch from Docker to Apple Container for macOS-native container isolation. Use when the user wants Apple Container instead of Docker, or is setting up on macOS and prefers the native runtime. Triggers on "apple container", "convert to apple container", "switch to apple container", or "use apple container".
---

# Convert to Apple Container

This skill switches ClaudeClaw's container runtime from Docker to Apple Container (macOS-only). It uses the skills engine for deterministic code changes, then walks through verification.

**What this changes:**
- Container runtime binary: `docker` → `container`
- Mount syntax: `-v path:path:ro` → `--mount type=bind,source=...,target=...,readonly`
- Startup check: `docker info` → `container system status` (with auto-start)
- Orphan detection: `docker ps --filter` → `container ls --format json`
- Build script default: `docker` → `container`
- Container runner: main-group containers **do not** mount the host project root (the kata kernel rejects bind-mounts over files inside a readonly mount, so the `.env` shadow trick from earlier versions of this skill cannot work). The agent loses read access to host source code in main groups; users who need it can re-add the path via the mount allowlist. Docker behavior is unchanged — the project root is mounted readonly and `.env` is shadowed via a host-side `/dev/null` bind mount.

**What stays the same:**
- Mount security/allowlist validation
- All exported interfaces and IPC protocol
- Non-main container behavior (still uses `--user` flag, gets group folder + read-only global memory)
- All other functionality

## Prerequisites

Verify Apple Container is installed:

```bash
container --version && echo "Apple Container ready" || echo "Install Apple Container first"
```

If not installed, prefer Homebrew (the project ships as a formula now, not a cask):

```bash
brew install container
container system start    # one-time, accepts the kata kernel install prompt
```

Or download from https://github.com/apple/container/releases and install the `.pkg`.

On Apple Silicon, the buildkit also requires Rosetta 2:

```bash
softwareupdate --install-rosetta --agree-to-license
```

Apple Container requires macOS. It does not work on Linux.

## Phase 1: Pre-flight

### Check if already applied

The runtime files moved from `src/orchestrator/` to `src/runtimes/` in a refactor. Check both locations:

```bash
grep -h "CONTAINER_RUNTIME_BIN" src/runtimes/container-runtime.ts src/orchestrator/container-runtime.ts 2>/dev/null
```

If the result shows `'container'`, the runtime is already Apple Container — skip to Phase 3. If the file is missing entirely, fall back to the old skill-branch merge in Phase 2.

## Phase 2: Apply Code Changes

### Ensure upstream remote

```bash
git remote -v
```

If `upstream` is missing, add it:

```bash
git remote add upstream https://github.com/sbusso/claudeclaw.git
```

### Merge the skill branch

```bash
git fetch upstream skill/apple-container
git merge upstream/skill/apple-container
```

This merges in:
- `src/runtimes/container-runtime.ts` — Apple Container implementation (replaces Docker; older trees may have this at `src/orchestrator/container-runtime.ts`)
- `src/runtimes/container-runtime.test.ts` — Apple Container-specific tests
- `src/runtimes/container-runner.ts` — main-group containers skip the host project-root mount on Apple Container (the kata kernel rejects bind-mounts inside readonly mounts, so `.env` cannot be shadowed there)
- `src/runtimes/docker/Dockerfile` — entrypoint shadows `.env` via `mount --bind /dev/null` only when the project mount is present (Docker only)
- `src/runtimes/docker/build.sh` — default runtime set to `container`

If the merge reports conflicts, resolve them by reading the conflicted files and understanding the intent of both sides.

### Validate code changes

```bash
npm test
npm run build
```

All tests must pass and build must be clean before proceeding.

## Phase 3: Verify

### Ensure Apple Container runtime is running

```bash
container system status || container system start
```

### Build the container image

```bash
./src/runtimes/docker/build.sh
```

### Test basic execution

```bash
echo '{}' | container run -i --entrypoint /bin/echo claudeclaw-agent:latest "Container OK"
```

### Test readonly mounts

```bash
mkdir -p /tmp/test-ro && echo "test" > /tmp/test-ro/file.txt
container run --rm --entrypoint /bin/bash \
  --mount type=bind,source=/tmp/test-ro,target=/test,readonly \
  claudeclaw-agent:latest \
  -c "cat /test/file.txt && touch /test/new.txt 2>&1 || echo 'Write blocked (expected)'"
rm -rf /tmp/test-ro
```

Expected: Read succeeds, write fails with "Read-only file system".

### Test read-write mounts

```bash
mkdir -p /tmp/test-rw
container run --rm --entrypoint /bin/bash \
  -v /tmp/test-rw:/test \
  claudeclaw-agent:latest \
  -c "echo 'test write' > /test/new.txt && cat /test/new.txt"
cat /tmp/test-rw/new.txt && rm -rf /tmp/test-rw
```

Expected: Both operations succeed.

> **Service name:** Derived from the directory name: `com.claudeclaw.<dirname>` (macOS) / `claudeclaw-<dirname>` (Linux). For example, if cwd is `my-assistant`, the service is `com.claudeclaw.my-assistant`. Determine the correct service name before running service commands below.

### Full integration test

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.claudeclaw
```

Send a message via WhatsApp and verify the agent responds.

## Troubleshooting

**Apple Container not found:**
- Download from https://github.com/apple/container/releases
- Install the `.pkg` file
- Verify: `container --version`

**Runtime won't start:**
```bash
container system start
container system status
```

**Image build fails:**
```bash
# Clean rebuild — Apple Container caches aggressively
container builder stop && container builder rm && container builder start
./src/runtimes/docker/build.sh
```

**Container can't write to mounted directories:**
Check directory permissions on the host. The container runs as uid 1000 in non-main groups; main groups start as root and drop privileges via `setpriv` to the host user (`RUN_UID`/`RUN_GID`).

**Build fails with "Rosetta is not installed":**
The Apple Container buildkit needs Rosetta on Apple Silicon. Install it and reset the buildkit:
```bash
softwareupdate --install-rosetta --agree-to-license
container builder stop && container builder rm && container builder start
```

**Container exits with `mount: /workspace/project/.env: mount point is not a directory`:**
This is the legacy bug that drove the current design. The kata kernel won't bind-mount over a regular file inside a readonly mount, regardless of source type (`/dev/null`, empty file, etc.). If you see it, you're running an older `container-runner.ts` that still mounts the host project root on Apple Container — pull the latest `src/runtimes/container-runner.ts`, which skips the mount on Apple Container entirely. Rebuild the image afterward.

## Summary of Changed Files

| File | Type of Change |
|------|----------------|
| `src/runtimes/container-runtime.ts` | Docker → Apple Container API (binary, mount syntax, startup, orphan detection). Older trees may have this at `src/orchestrator/container-runtime.ts`. |
| `src/runtimes/container-runtime.test.ts` | Tests for Apple Container behavior |
| `src/runtimes/container-runner.ts` | Main-group containers skip the host project-root mount on Apple Container (kata kernel can't shadow `.env` inside a readonly mount). Containers still start as root and drop privileges via `setpriv`. |
| `src/runtimes/docker/Dockerfile` | Entrypoint shadows `.env` via `mount --bind /dev/null` only when present (i.e. on Docker — Apple Container no longer mounts the project root) and drops privileges via `setpriv`. |
| `src/runtimes/docker/build.sh` | Default runtime: `docker` → `container` |

**Trade-off note:** On Apple Container, the agent in main groups can no longer read host source under `/workspace/project`. If a use case requires that access (e.g. self-improvement workflows that edit ClaudeClaw's own code), add the project root to the user's mount allowlist explicitly — accepting that `.env` will then be readable too.
