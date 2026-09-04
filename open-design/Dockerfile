# Open Design + Claude Code (subscription auth)
#
# The published od image deliberately ships without any agent CLI. This layer
# adds Claude Code so the daemon can auto-detect it in PATH at startup.
#
# No ANTHROPIC_API_KEY anywhere: auth comes from the OAuth credentials in
# $HOME/.claude, which you create once with `claude` -> /login (see README).

FROM ghcr.io/nexu-io/od:latest

USER root

# libc6-compat: the Claude Code native binary is glibc-linked, base image is Alpine.
# git: Claude Code expects it for repo-aware work.
# ripgrep: Claude Code's file search; without it, search falls back and is slow.
RUN apk add --no-cache libc6-compat curl bash git ripgrep ca-certificates

# Fixed home outside /app so it can't collide with the daemon's data contract
# (/app/.od). Claude Code resolves ~/.claude from HOME, and CLAUDE_CONFIG_DIR
# has a known path-expansion bug, so set HOME rather than that variable.
ENV HOME=/opt/agent
ENV PATH=/opt/agent/.local/bin:$PATH

RUN mkdir -p /opt/agent/.claude \
 && curl -fsSL https://claude.ai/install.sh | bash \
 && chown -R 1001:1001 /opt/agent

# Base image runs unprivileged as uid 1001; the data volume expects that owner.
USER 1001

# Fail the build early if the CLI didn't land or won't execute on Alpine.
RUN claude --version
