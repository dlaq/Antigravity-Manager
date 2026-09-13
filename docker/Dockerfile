# --- Upstream Source Stage ---
FROM alpine:3.20 AS source
RUN apk add --no-cache git ca-certificates
RUN git clone --depth 1 --branch v4.7.1 https://github.com/lbjlaq/Antigravity-Manager.git /src
COPY patches/chat-completions-assistant-prefill.patch /patches/chat-completions-assistant-prefill.patch
COPY patches/gemini-penalty-compat.patch /patches/gemini-penalty-compat.patch
RUN cd /src \
    && git apply --check /patches/chat-completions-assistant-prefill.patch \
    && git apply /patches/chat-completions-assistant-prefill.patch \
    && git apply --check /patches/gemini-penalty-compat.patch \
    && git apply /patches/gemini-penalty-compat.patch \
    && grep -Fq 'rewrite_terminal_assistant_prefill(messages)' src-tauri/src/proxy/handlers/openai.rs \
    && grep -Fq '[Chat Completions Compat] Rewrote terminal plain-text assistant prefill' src-tauri/src/proxy/handlers/openai.rs \
    && grep -Fq 'is_native_gemini_model' src-tauri/src/proxy/mappers/openai/request.rs \
    && grep -Fq '[OpenAI-Compat] Dropped presence/frequency penalty for Gemini model' src-tauri/src/proxy/mappers/openai/request.rs

# --- Frontend Build Stage ---
FROM node:20-slim AS frontend-builder
ARG USE_MIRROR=auto
WORKDIR /app

RUN if [ "$USE_MIRROR" = "true" ] || ( [ "$USE_MIRROR" = "auto" ] && ! timeout 3 bash -c "</dev/tcp/www.google.com/80" 2>/dev/null ); then \
    echo "Using npm mirror..."; \
    npm config set registry https://registry.npmmirror.com; \
    else \
    echo "Using default npm registry..."; \
    fi

COPY --from=source /src/package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --legacy-peer-deps --prefer-offline --no-audit --no-fund
COPY --from=source /src/tsconfig.json /src/tsconfig.node.json /src/vite.config.ts /src/tailwind.config.js /src/postcss.config.cjs /src/index.html ./
COPY --from=source /src/public ./public
COPY --from=source /src/src ./src
COPY --from=source /src/src-tauri/icons ./src-tauri/icons
RUN npm run build

# --- Backend Build Stage ---
FROM rust:1-slim-bookworm AS backend-builder
ARG USE_MIRROR=auto

RUN if [ "$USE_MIRROR" = "true" ] || ( [ "$USE_MIRROR" = "auto" ] && ! timeout 3 bash -c "</dev/tcp/www.google.com/80" 2>/dev/null ); then \
    echo "Using Aliyun mirror for APT..."; \
    sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list.d/debian.sources || \
    sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list; \
    else \
    echo "Using default APT sources..."; \
    fi

RUN apt-get update && apt-get install -y \
    pkg-config \
    build-essential \
    curl \
    wget \
    file \
    libssl-dev \
    libgtk-3-dev \
    libwebkit2gtk-4.1-dev \
    libayatana-appindicator3-dev \
    librsvg2-dev \
    libsoup-3.0-dev \
    libjavascriptcoregtk-4.1-dev \
    perl \
    cmake \
    golang-go \
    clang \
    libclang-dev \
    git \
    && rm -rf /var/lib/apt/lists/*

ENV CARGO_HTTP_MULTIPLEXING=false
RUN if [ "$USE_MIRROR" = "true" ] || ( [ "$USE_MIRROR" = "auto" ] && ! timeout 3 bash -c "</dev/tcp/www.google.com/80" 2>/dev/null ); then \
    echo "Using Aliyun mirror for Cargo..."; \
    mkdir -p /root/.cargo && \
    echo "[source.crates-io]\nreplace-with = 'aliyun'\n\n[source.aliyun]\nregistry = \"sparse+https://mirrors.aliyun.com/crates.io-index/\"" > /root/.cargo/config.toml; \
    else \
    echo "Using default Cargo registry..."; \
    fi

RUN which go && go version || (echo "ERROR: Go compiler not found" && exit 1)

WORKDIR /app
COPY --from=source /src/src-tauri ./src-tauri
COPY --from=source /src/src/locales ./src/locales
COPY --from=frontend-builder /app/dist ./dist

WORKDIR /app/src-tauri
RUN --mount=type=cache,target=/root/.cargo/registry \
    --mount=type=cache,target=/root/.cargo/git \
    --mount=type=cache,target=/app/src-tauri/target \
    cargo build --release --bin antigravity-tools && \
    cp target/release/antigravity-tools /tmp/antigravity-tools

# --- Final Runtime Stage ---
FROM debian:bookworm-slim
ARG USE_MIRROR=auto
WORKDIR /app

RUN if [ "$USE_MIRROR" = "true" ] || ( [ "$USE_MIRROR" = "auto" ] && ! timeout 3 bash -c "</dev/tcp/www.google.com/80" 2>/dev/null ); then \
    echo "Using Aliyun mirror for APT..."; \
    sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list.d/debian.sources || \
    sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list; \
    else \
    echo "Using default APT sources..."; \
    fi

RUN apt-get update && apt-get install -y \
    libssl3 \
    libsqlite3-0 \
    ca-certificates \
    libgtk-3-0 \
    libwebkit2gtk-4.1-0 \
    libayatana-appindicator3-1 \
    librsvg2-2 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=backend-builder /tmp/antigravity-tools /app/antigravity-tools
COPY --from=frontend-builder /app/dist /app/dist

ENV ABV_DIST_PATH=/app/dist
ENV RUST_LOG=info
ENV PORT=8045

EXPOSE 8045
ENTRYPOINT ["/app/antigravity-tools", "--headless"]
