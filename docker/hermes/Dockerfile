FROM nousresearch/hermes-agent:latest

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates gnupg \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && npm install -g pnpm \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/tdai-memory-openclaw-plugin

RUN printf '{\n  "name": "tdai-memory-hermes-runtime",\n  "private": true,\n  "type": "module"\n}\n' > package.json \
    && npm install --omit=dev @tencentdb-agent-memory/memory-tencentdb@latest tsx \
    && npm cache clean --force

COPY docker/hermes/hermes-stack-entrypoint /usr/local/bin/hermes-stack-entrypoint

RUN chmod 755 /usr/local/bin/hermes-stack-entrypoint
