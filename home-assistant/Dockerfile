
# syntax=docker/dockerfile:1

FROM ghcr.io/home-assistant/home-assistant:latest

# Metadata (good hygiene)
LABEL org.opencontainers.image.title="Home Assistant"
LABEL org.opencontainers.image.description="Production-ready Home Assistant container"
LABEL org.opencontainers.image.source="https://www.home-assistant.io/"
LABEL org.opencontainers.image.version="2024.12.1"

# Home Assistant runs as root internally (required for some integrations)
# Do NOT change USER unless you know exactly which integrations you use.

# Healthcheck – Docker & orchestrators can detect failures
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8123/ || exit 1

# Expose HA UI
EXPOSE 8123

# Config directory (volume is declared here for clarity)
VOLUME ["/config"]

# Entrypoint is already defined by the base image
