# syntax=docker/dockerfile:1
#
# Open Design + Claude Code (subscription auth), headless-ready:
# HyperFrames video rendering, browser sessions / web-clone, and the shell
# tools the bundled skills call.
#
# The published od image deliberately ships without any agent CLI. This layer
# adds Claude Code so the daemon can auto-detect it in PATH at startup.
#
# No ANTHROPIC_API_KEY anywhere: auth comes from the OAuth credentials in
# $HOME/.claude, which you create once with `claude` -> /login (see README).
#
# Needs from the runtime (compose), which this file cannot set:
#   - a volume on /opt/agent           login, ~/.claude.json, caches
#   - memory >= 2g (4g recommended)    upstream's 384m only fits the idle daemon
#   - pids limit >= 1024               Chromium + Claude Code exceed upstream's 256
#   - NODE_OPTIONS: if your compose sets it (upstream's does, to 192 MB),
#     raise it there too; compose environment overrides the ENV below.

ARG OD_IMAGE=ghcr.io/nexu-io/od:latest
FROM ${OD_IMAGE}

# npm dist-tag (stable | latest) or an exact version such as 2.1.287.
# Pin an exact version (and OD_IMAGE by digest) for reproducible builds.
ARG CLAUDE_CODE_VERSION=stable

# Extra apk packages without editing this file, e.g.
#   --build-arg EXTRA_APK_PACKAGES="font-noto-arabic libc6-compat"
ARG EXTRA_APK_PACKAGES=""

USER root

RUN apk add --no-cache \
      # Claude Code on Alpine. It ships a native musl build, so no glibc shim
      # (libc6-compat) is needed. libgcc/libstdc++ are its documented runtime
      # deps; ripgrep replaces the bundled one (see USE_BUILTIN_RIPGREP below).
      bash ca-certificates git libgcc libstdc++ ripgrep \
      # Tools Open Design's prompts and bundled skills run from the agent shell:
      # python3 parses the media-generation JSON, openssl names HyperFrames
      # compositions, unzip/zip unpack office files, gh backs GitHub features.
      curl github-cli jq openssl py3-pillow python3 unzip zip \
      # Headless rendering. chromium-swiftshader is the software WebGL backend
      # HyperFrames needs for shader transitions when there is no GPU. ffmpeg
      # also provides ffprobe, which the daemon's bundled ffmpeg does not.
      chromium chromium-swiftshader ffmpeg \
      # Fonts, or text renders as boxes in videos and screenshots.
      # font-noto-cjk is the big one (~90 MB); drop it if you never render CJK.
      fontconfig font-inter font-liberation font-noto font-noto-cjk font-noto-emoji \
      ${EXTRA_APK_PACKAGES} \
 && fc-cache -f

# Same native binary as claude.ai/install.sh, installed through npm so that it
# lands in /usr/local: root-owned, version-pinnable, and outside HOME. That
# leaves HOME as pure state, so all of it can be one volume without a stale
# binary in the volume shadowing a rebuilt image.
RUN npm install -g --no-audit --no-fund --cache /tmp/npm-cache \
      "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" \
 && rm -rf /tmp/npm-cache

# Fixed home outside /app so it can't collide with the daemon's data contract
# (/app/.od). Claude Code resolves ~/.claude and ~/.claude.json from HOME, and
# CLAUDE_CONFIG_DIR has a known path-expansion bug, so set HOME rather than
# that variable.
ENV HOME=/opt/agent

# The image is immutable: upgrade Claude Code by rebuilding, not in place.
# Alpine: use the system ripgrep instead of the bundled one.
ENV DISABLE_AUTOUPDATER=1 \
    USE_BUILTIN_RIPGREP=0

# HyperFrames. Without the desktop app the daemon only renders video when
# HYPERFRAMES_BROWSER_PATH is set; otherwise every render fails with "desktop
# frame renderer is unavailable". Full Chromium has no BeginFrame control, so
# force screenshot capture instead of relying on the runtime probe. Keep the
# extracted-frame cache on the HOME volume instead of /tmp (tmpfs upstream).
ENV HYPERFRAMES_BROWSER_PATH=/usr/bin/chromium \
    PRODUCER_FORCE_SCREENSHOT=true \
    HYPERFRAMES_EXTRACT_CACHE_DIR=/opt/agent/.cache/hyperframes/extract \
    HYPERFRAMES_NO_UPDATE_CHECK=1

# Browser sessions and web-clone launch the system browser themselves.
# Chromium's own sandbox needs user namespaces, which Docker's default seccomp
# profile blocks, so the container is the isolation boundary here. Alpine's
# launcher appends CHROMIUM_USER_FLAGS to every start.
ENV OD_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium \
    CHROMIUM_USER_FLAGS="--no-sandbox --disable-dev-shm-usage"

# The base image caps the Node heap at 192 MB, and the HyperFrames CLI the
# daemon spawns inherits that cap.
ENV NODE_OPTIONS=--max-old-space-size=1024

RUN mkdir -p /opt/agent/.claude /opt/agent/.cache \
 && chown -R 1001:1001 /opt/agent

# Base image runs unprivileged as uid 1001; the data volume expects that owner.
# It also matters for Claude Code: Open Design starts it with
# --permission-mode bypassPermissions, which Claude Code refuses to run as root.
USER 1001

# Fail the build early if any piece didn't land or won't execute on Alpine.
# No `| head` here: sh has no pipefail, so a pipe would hide a failing command.
RUN claude --version \
 && rg --version \
 && chromium --version \
 && ffprobe -version > /dev/null \
 && ffmpeg -hide_banner -encoders | grep -q libx264 \
 && node -p "'hyperframes ' + require('/app/apps/daemon/node_modules/hyperframes/package.json').version"

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -q -O /dev/null "http://127.0.0.1:${OD_PORT:-7456}/api/health" || exit 1