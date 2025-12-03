# Example multi-stage Dockerfile
# Rename to Dockerfile and customize for your application

FROM debian:bookworm-slim AS base

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=C.UTF-8

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*

ARG USERNAME=app
ARG USER_UID=1000
ARG USER_GID=1000

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME

# Minimal target - lightweight image
FROM base AS minimal

WORKDIR /app
USER app
CMD ["bash"]

# Runtime target - full application
FROM base AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    jq \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
USER app
CMD ["bash"]
