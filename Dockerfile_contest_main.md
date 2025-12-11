# Top Dockerfile REACT
## Dockerfile build Docker Image nhẹ nhất (REACT)
```
# syntax=docker/dockerfile:1.7
# ==============================================================================
# Build Stage - Using Node Alpine for minimal size
# ==============================================================================
FROM node:22.21.1-alpine3.21@sha256:af8023ec879993821f6d5b21382ed915622a1b0f1cc03dbeb6804afaf01f8885 AS builder

# Install pnpm with specific version from package.json and gzip for pre-compression
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && \
  corepack prepare pnpm --activate

WORKDIR /app

# Copy package files for dependency installation (optimized layer caching)
COPY package.json pnpm-lock.yaml ./

# Install dependencies with cache mount for faster rebuilds
# Installs all dependencies (including devDependencies needed for build: typescript, vite, tailwindcss, etc.)
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
  pnpm install --frozen-lockfile

# Copy only necessary source files (exclude tests, docs, config files not needed for build)
COPY tsconfig.json tsconfig.node.json vite.config.ts tailwind.config.ts postcss.config.js ./
COPY index.html ./
COPY public ./public
COPY src ./src

# Build the application
RUN pnpm run build && \
  # Verify build output exists
  test -d dist && test -f dist/index.html && \
  # Remove bundle visualizer output (not needed in production, saves ~100KB compressed)
  rm -f dist/stats.html && \
  # Create a minimal health check endpoint (1 byte file for ultra-fast response)
  echo "OK" > dist/health && \
  # Pre-compress all static files with gzip (level 9 = maximum compression)
  find dist -type f \( \
  -name "*.html" -o \
  -name "*.css" -o \
  -name "*.js" -o \
  -name "*.json" -o \
  -name "*.xml" -o \
  -name "*.txt" -o \
  -name "*.svg" \
  \) -exec sh -c 'gzip -9 "{}"' \;

# ==============================================================================
# Production Stage - Using lipanski/docker-static-website for extreme minimal footprint (92.5 KB base)
# ==============================================================================
FROM lipanski/docker-static-website:latest AS production

# Add OCI labels for metadata
LABEL org.opencontainers.image.title="Vite React Template" \
  org.opencontainers.image.description="Production-ready Vite React application with extreme minimal footprint" \
  org.opencontainers.image.version="0.4.0" \
  org.opencontainers.image.licenses="MIT OR Apache-2.0" \
  org.opencontainers.image.base.name="lipanski/docker-static-website:latest"

# Copy built assets from builder stage
# lipanski/docker-static-website serves from /home/static
COPY --from=builder /app/dist /home/static

# Expose port (BusyBox httpd uses port 3000 by default)
EXPOSE 3000

# The base image already has CMD set to run BusyBox httpd
# It automatically serves .gz files when Accept-Encoding: gzip is present
# No additional configuration needed - inherited from base image
```
## Dockerfile TOP 1 (REACT)
```
# syntax=docker/dockerfile:1.7

# Multi-arch support: Automatically provided by buildx
ARG BUILDPLATFORM
ARG TARGETPLATFORM
ARG TARGETARCH

ARG BUILD_DATE
ARG GIT_COMMIT=unknown
ARG NGINX_VERSION=1.26.2
ARG NODE_VERSION=20
ARG ALPINE_VERSION=3.20

# ==============================================================================
# Stage 1: Application Build
# ==============================================================================
FROM node:${NODE_VERSION}-alpine@sha256:2d5e8a8a51bc341fd5f2eed6d91455c3a3d147e91a14298fc564b5dc519c1666 AS builder

WORKDIR /app

# Setup pnpm with corepack
ENV PNPM_HOME="/pnpm" \
    PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@9.12.2 --activate

# Install dependencies with cache mount
COPY package.json pnpm-lock.yaml .npmrc ./
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile --prefer-offline

# Copy source and build configuration
COPY tsconfig.json tsconfig.node.json vite.config.ts ./
COPY postcss.config.js tailwind.config.ts biome.json ./
COPY index.html ./
COPY public ./public
COPY src ./src

# Build and clean artifacts
ENV NODE_ENV=production
RUN pnpm build && \
    find dist -type f \( -name "*.map" -o -name ".*" \) -delete && \
    rm -f dist/stats.html

# ==============================================================================
# Stage 2: Static Nginx Binary Builder
# ==============================================================================
FROM alpine:${ALPINE_VERSION}@sha256:765942a4039992336de8dd5db680586e1a206607dd06170ff0a37267a9e01958 AS nginx-builder

ARG NGINX_VERSION
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ENV NGINX_SHA256=627fe086209bba80a2853a0add9d958d7ebbdffa1a8467a5784c9a6b4f03d738

# Log build platform info for multi-arch
RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM"

# Install build dependencies
RUN apk add --no-cache \
    gcc g++ musl-dev make linux-headers curl \
    pcre-dev pcre2-dev zlib-dev zlib-static \
    openssl-dev openssl-libs-static upx

# Download and verify nginx
WORKDIR /tmp
RUN curl -fSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" -o nginx.tar.gz && \
    echo "${NGINX_SHA256}  nginx.tar.gz" | sha256sum -c -

# Build fully static nginx with minimal modules
RUN tar -xzf nginx.tar.gz && \
    cd "nginx-${NGINX_VERSION}" && \
    ./configure \
        --prefix=/usr/local/nginx \
        --sbin-path=/usr/local/nginx/sbin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --pid-path=/run/nginx.pid \
        --lock-path=/run/nginx.lock \
        --error-log-path=/dev/stderr \
        --http-log-path=/dev/stdout \
        --user=nobody \
        --group=nobody \
        # Performance features
        --with-threads \
        --with-file-aio \
        --with-http_ssl_module \
        --with-http_v2_module \
        --with-http_gzip_static_module \
        --with-http_stub_status_module \
        --with-pcre \
        --with-pcre-jit \
        # Static linking and optimization
        --with-cc-opt='-static -Os -ffunction-sections -fdata-sections' \
        --with-ld-opt='-static -Wl,--gc-sections' \
        # Disable unnecessary modules
        --without-http_charset_module \
        --without-http_ssi_module \
        --without-http_userid_module \
        --without-http_auth_basic_module \
        --without-http_mirror_module \
        --without-http_autoindex_module \
        --without-http_geo_module \
        --without-http_map_module \
        --without-http_split_clients_module \
        --without-http_referer_module \
        --without-http_rewrite_module \
        --without-http_proxy_module \
        --without-http_fastcgi_module \
        --without-http_uwsgi_module \
        --without-http_scgi_module \
        --without-http_grpc_module \
        --without-http_memcached_module \
        --without-http_limit_conn_module \
        --without-http_limit_req_module \
        --without-http_empty_gif_module \
        --without-http_browser_module \
        --without-http_upstream_hash_module \
        --without-http_upstream_ip_hash_module \
        --without-http_upstream_least_conn_module \
        --without-http_upstream_random_module \
        --without-http_upstream_keepalive_module \
        --without-http_upstream_zone_module \
        --without-mail_pop3_module \
        --without-mail_imap_module \
        --without-mail_smtp_module \
        --without-stream_limit_conn_module \
        --without-stream_access_module \
        --without-stream_geo_module \
        --without-stream_map_module \
        --without-stream_split_clients_module \
        --without-stream_return_module \
        --without-stream_set_module \
        --without-stream_upstream_hash_module \
        --without-stream_upstream_least_conn_module \
        --without-stream_upstream_random_module \
        --without-stream_upstream_zone_module && \
    make -j"$(nproc)" && \
    make install

# Optimize binary: strip symbols + UPX compression
RUN strip --strip-all /usr/local/nginx/sbin/nginx && \
    upx --best --lzma /usr/local/nginx/sbin/nginx && \
    /usr/local/nginx/sbin/nginx -V

# ==============================================================================
# Stage 3: Asset Compression
# ==============================================================================
FROM alpine:${ALPINE_VERSION}@sha256:765942a4039992336de8dd5db680586e1a206607dd06170ff0a37267a9e01958 AS compressor

RUN apk add --no-cache brotli gzip findutils

WORKDIR /app
COPY --from=builder /app/dist ./dist

# Parallel compression: gzip + brotli for all text-based assets
RUN find dist -type f \
    \( -name "*.html" -o -name "*.css" -o -name "*.js" -o \
       -name "*.json" -o -name "*.svg" -o -name "*.xml" \) \
    -print0 | xargs -0 -P"$(nproc)" -I {} sh -c 'gzip -9 -k -f "{}" && brotli -q 11 -f "{}"'

# ==============================================================================
# Stage 4: Minimal Filesystem Preparation
# ==============================================================================
FROM alpine:${ALPINE_VERSION}@sha256:765942a4039992336de8dd5db680586e1a206607dd06170ff0a37267a9e01958 AS rootfs

# Create directory structure
RUN mkdir -p \
    /rootfs/etc/nginx/conf.d \
    /rootfs/usr/share/nginx/html \
    /rootfs/var/log/nginx \
    /rootfs/var/cache/nginx \
    /rootfs/usr/local/nginx/{client_body,proxy,fastcgi,uwsgi,scgi}_temp \
    /rootfs/tmp \
    /rootfs/run && \
    chmod 1777 /rootfs/tmp

# Create minimal user database (nobody user)
RUN echo "nobody:x:65534:65534:nobody:/:/sbin/nologin" > /rootfs/etc/passwd && \
    echo "nobody:x:65534:" > /rootfs/etc/group

# Copy nginx configuration
COPY nginx.conf /rootfs/etc/nginx/conf.d/default.conf
COPY --from=nginx-builder /etc/nginx/mime.types /rootfs/etc/nginx/mime.types
COPY --from=compressor /app/dist /rootfs/usr/share/nginx/html

# Create main nginx.conf
RUN cat > /rootfs/etc/nginx/nginx.conf <<'EOF'
worker_processes auto;
error_log stderr warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    access_log /dev/stdout;

    # Performance optimizations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    server_tokens off;

    # Compression
    gzip on;
    gzip_static on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;

    include /etc/nginx/conf.d/*.conf;
}
EOF

# Set proper ownership
RUN chown -R 65534:65534 \
    /rootfs/usr/share/nginx/html \
    /rootfs/var/log/nginx \
    /rootfs/var/cache/nginx \
    /rootfs/usr/local/nginx \
    /rootfs/tmp \
    /rootfs/run

# ==============================================================================
# Stage 5: Final Distroless Image (FROM SCRATCH)
# ==============================================================================
FROM scratch

# Re-declare build args for metadata
ARG BUILD_DATE
ARG GIT_COMMIT=unknown

# OCI metadata labels
LABEL org.opencontainers.image.title="Vite React - Distroless" \
      org.opencontainers.image.description="Distroless minimal image (<6MB) - UPX compressed" \
      org.opencontainers.image.version="2.2.0-distroless-upx" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.base.name="scratch" \
      org.opencontainers.image.source="https://github.com/riipandi/vite-react-template" \
      org.opencontainers.image.licenses="MIT OR Apache-2.0" \
      maintainer="contest-2025-optimized"

# Copy static nginx binary and minimal filesystem
COPY --from=nginx-builder /usr/local/nginx/sbin/nginx /usr/sbin/nginx
COPY --from=rootfs /rootfs /

# Run as non-root user (nobody = 65534)
USER 65534:65534

EXPOSE 3000

# Lightweight healthcheck using nginx config test
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/usr/sbin/nginx", "-t", "-q"]

STOPSIGNAL SIGTERM

ENTRYPOINT ["/usr/sbin/nginx"]
CMD ["-g", "daemon off;"]
```
## Dockerfile TOP 2 (REACT)
```
# STAGE 1: The Builder (Custom Nginx Build) 
FROM alpine:3.19 AS builder

# Multi-arch support
ARG TARGETARCH
RUN apk add --no-cache build-base pcre2-dev zlib-dev openssl-dev

# Download and verify Nginx source with SHA256 checksum
ARG NGINX_VERSION=1.27.0
ARG NGINX_SHA256=b7230e3cf87eaa2d4b0bc56aadc920a960c7873b9991a1b66ffcc08fc650129c
ADD --checksum=sha256:${NGINX_SHA256} https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz /tmp/
RUN tar -xzf /tmp/nginx-${NGINX_VERSION}.tar.gz -C /tmp

# Configure Nginx with minimal modules (static file serving only)
RUN cd /tmp/nginx-${NGINX_VERSION} && \
    ./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --pid-path=/var/run/nginx.pid \
    --error-log-path=/dev/stderr \
    --http-log-path=/dev/stdout \
    --user=nginx \
    --group=nginx \
    --without-http_rewrite_module \
    --without-http_gzip_module \
    --without-http_proxy_module \
    --without-http_fastcgi_module \
    --without-http_uwsgi_module \
    --without-http_scgi_module \
    --without-mail_pop3_module \
    --without-mail_imap_module \
    --without-mail_smtp_module && \
    make && \
    make install && rm -rf /tmp/* && strip /usr/sbin/nginx

# Create file Nginx main config (include snippet file)
RUN { \
    mkdir -p /etc/nginx/conf.d && \
    cat > /etc/nginx/nginx.conf; \
    } <<EOF
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    client_body_temp_path /var/cache/nginx/client_body_temp;
    include       /etc/nginx/conf.d/default.conf;
}
EOF

RUN mkdir -p /etc/nginx/conf.d && \
    cat > /etc/nginx/mime.types <<'EOF'
types {
    text/html    html;
    text/css     css;
    application/javascript  js;
    image/png    png;
    application/json json;
}
EOF

# Create minimal user/group files (no full /etc/passwd needed in scratch image)
RUN echo "nginx:x:101:101:nginx:/var/cache/nginx:/sbin/nologin" > /etc/passwd && \
    echo "nginx:x:101:" > /etc/group

# Build static healthcheck binary (no external dependencies like wget/curl needed)
RUN { \
    cat > /tmp/healthcheck.c && \
    gcc -static -O2 -o /healthcheck /tmp/healthcheck.c; \
    } <<EOF
#include <netdb.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    struct hostent *h = gethostbyname("localhost");
    if (!h) return 1;

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) return 1;

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(3000);
    addr.sin_addr = *(struct in_addr *)h->h_addr_list[0];

    int result = connect(sock, (struct sockaddr *)&addr, sizeof(addr));
    close(sock);
    return (result == 0) ? 0 : 1;
}
EOF

# Collect shared libraries required by Nginx (supports multi-arch)
RUN mkdir -p /staging/lib /staging/usr/lib && \
    ldd /usr/sbin/nginx | tr -s '[:space:]' '\n' | grep '^/' | \
    xargs -I '{}' sh -c 'mkdir -p /staging$(dirname {}) && cp -L {} /staging$(dirname {})'

# Create mount point directories for tmpfs (writable dirs in read-only container)
RUN mkdir -p /var/cache/nginx /var/run /tmp && chown -R 101:101 /var/cache/nginx /var/run /tmp

# STAGE 2: App Builder (Build React App) 
FROM node:20.11.0-alpine3.19 AS app_builder

WORKDIR /app
RUN corepack enable && corepack prepare pnpm@9.12.2 --activate

# Copy dependencies files first for caching
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store,sharing=locked \
    pnpm install --frozen-lockfile --prefer-offline

# Copy config files
COPY tsconfig.json tsconfig.node.json vite.config.ts ./
COPY postcss.config.js tailwind.config.ts index.html ./

# Copy source code (avoid copying tests, stories, etc.)
COPY public/ ./public/
COPY src/ ./src/

RUN pnpm build && \
    test -f dist/index.html || (echo "Build failed" && exit 1)

# STAGE 3: Production Image (FROM scratch) 
FROM scratch

LABEL org.opencontainers.image.title="Vite React Template" \
    org.opencontainers.image.description="Minimal Vite React SPA with custom Nginx built from scratch" \
    org.opencontainers.image.version="1.0.0" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.base.name="scratch" \
    org.opencontainers.image.authors="Contest 2025"

# Copy file user/group
COPY --from=builder /etc/passwd /etc/group /etc/

# Copy mount points to image
COPY --chown=101:101 --from=builder /var/cache/nginx /var/cache/nginx
COPY --chown=101:101 --from=builder /var/run /var/run
COPY --chown=101:101 --from=builder /tmp /tmp

# Copy Nginx and shared libraries
COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /staging/ /

# Copy main Nginx config (just created)
COPY --from=builder /etc/nginx/nginx.conf /etc/nginx/nginx.conf

# Copy file config from project into included location
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy remaining necessary files
COPY --from=builder /etc/nginx/mime.types /etc/nginx/mime.types
COPY --from=app_builder /app/dist /usr/share/nginx/html
COPY --from=builder /healthcheck /healthcheck

USER 101:101

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD ["/healthcheck"]

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```
## Dockerfile TOP 3 (REACT)
```
# Stage 1: Build frontend
FROM node:20-alpine AS builder

RUN corepack enable pnpm

WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY tsconfig*.json vite.config.ts postcss.config.js tailwind.config.ts ./
COPY index.html ./
COPY src ./src
COPY public ./public

RUN pnpm build

# Clean up dist (remove source maps, licenses, stats, robots.txt, etc.)
RUN find /app/dist -type f -name '*.map' -delete \
 && find /app/dist -type f -name '*.LICENSE.*' -delete \
 && find /app/dist -type f -name '*.txt' -delete \
 && find /app/dist -type f -name 'stats.html' -delete \
 && find /app/dist -type f -name 'robots.txt' -delete \
 && find /app/dist -type f -name '_redirects' -delete

# Stage 2: Build Go server with FastHTTP
FROM golang:1.21-alpine AS go-builder

WORKDIR /app

# Cài strip và upx
RUN apk add --no-cache binutils upx

# Copy dist files để embed
COPY --from=builder /app/dist ./dist

# Tạo go.mod và main.go với FastHTTP
RUN cat > go.mod << 'GOMOD'
module server

go 1.21

require github.com/valyala/fasthttp v1.51.0
GOMOD

RUN cat > main.go << 'GOSRC'
package main

import (
    "embed"
    "log"
    "os"
    "path"
    "strings"

    "github.com/valyala/fasthttp"
)

//go:embed dist
var distFiles embed.FS

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "3000"
    }

    // FastHTTP handler
    handler := func(ctx *fasthttp.RequestCtx) {
        pathStr := string(ctx.Path())
        
        // Check if it's a static asset (has file extension)
        if strings.Contains(path.Base(pathStr), ".") {
            // Try to serve static file
            file, err := distFiles.Open("dist" + pathStr)
            if err == nil {
                defer file.Close()
                
                // Set content type based on extension
                ext := path.Ext(pathStr)
                switch ext {
                case ".js":
                    ctx.SetContentType("application/javascript")
                case ".css":
                    ctx.SetContentType("text/css")
                case ".svg":
                    ctx.SetContentType("image/svg+xml")
                case ".png":
                    ctx.SetContentType("image/png")
                case ".jpg", ".jpeg":
                    ctx.SetContentType("image/jpeg")
                case ".ico":
                    ctx.SetContentType("image/x-icon")
                default:
                    ctx.SetContentType("application/octet-stream")
                }
                
                // Copy file content to response
                ctx.Response.SetBodyStream(file, -1)
                return
            }
        }
        
        // SPA fallback - serve index.html for all routes
        file, err := distFiles.Open("dist/index.html")
        if err != nil {
            ctx.SetStatusCode(404)
            ctx.SetBodyString("Not Found")
            return
        }
        defer file.Close()
        
        ctx.SetContentType("text/html; charset=utf-8")
        ctx.Response.SetBodyStream(file, -1)
    }

    // healthcheck
    if len(os.Args) > 1 && os.Args[1] == "-health" {
        _, _, err := fasthttp.Get(nil, "http://127.0.0.1:"+port)
        if err != nil {
            os.Exit(1)
        }
        os.Exit(0)
    }

    log.Printf("FastHTTP server on :%s", port)
    log.Fatal(fasthttp.ListenAndServe(":"+port, handler))
}
GOSRC

# Download dependencies và build với tối ưu extreme
RUN go mod tidy
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -trimpath -o server main.go
RUN strip server
RUN upx --ultra-brute --lzma server

# Stage 3: Ultra-minimal runtime (scratch)
FROM scratch

# Copy binary (chứa cả static files)
COPY --from=go-builder /app/server /server

EXPOSE 3000

# Thêm USER 1000
USER 1000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/server", "-health"]

CMD ["/server"]
```
## Dockerfile TOP (REACT)
```
# =========================
# Giai đoạn 1: Node base
# =========================
# Môi trường build cho Vite/React: chỉ cài những thứ tối thiểu để biên dịch
FROM alpine:3.20@sha256:765942a4039992336de8dd5db680586e1a206607dd06170ff0a37267a9e01958 AS node-base
RUN apk add --no-cache nodejs npm git python3 g++ make && \
    npm install -g pnpm@9.12.2 && npm cache clean --force

# =========================
# Giai đoạn 2: Builder
# =========================
# Dùng cache mount cho pnpm để tăng tốc build lại; xóa source map; nén gzip sẵn
FROM node-base AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm-custom,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --prefer-offline
# Sao chép cấu hình & mã nguồn
COPY index.html ./
COPY vite.config.ts tsconfig.json tsconfig.node.json ./
COPY postcss.config.js tailwind.config.ts ./
COPY public ./public
COPY src ./src
# Build & tối ưu artefact tĩnh
RUN pnpm run build && \
    find /app/dist -name "*.map" -type f -delete || true && \
    find /app/dist -type f \( -name '*.html' -o -name '*.js' -o -name '*.css' -o -name '*.svg' -o -name '*.json' \) \
    -exec gzip -k -9 {} \;

# ======================================
# Giai đoạn 3: Nginx builder (tự biên dịch)
# ======================================
# Biên dịch nginx 1.27.3 với module tối thiểu cho SPA + nén tĩnh
FROM alpine:3.20@sha256:765942a4039992336de8dd5db680586e1a206607dd06170ff0a37267a9e01958 AS nginx-builder
RUN apk add --no-cache --virtual .build-deps gcc libc-dev make pcre2-dev zlib-dev openssl-dev linux-headers && \
    wget -O /tmp/nginx.tar.gz https://nginx.org/download/nginx-1.27.3.tar.gz && \
    tar -xzf /tmp/nginx.tar.gz -C /tmp && cd /tmp/nginx-1.27.3 && \
    ./configure \
      --prefix=/etc/nginx \
      --sbin-path=/usr/sbin/nginx \
      --modules-path=/usr/lib/nginx/modules \
      --conf-path=/etc/nginx/nginx.conf \
      --error-log-path=/var/log/nginx/error.log \
      --http-log-path=/var/log/nginx/access.log \
      --pid-path=/var/run/nginx.pid \
      --lock-path=/var/run/nginx.lock \
      --http-client-body-temp-path=/var/cache/nginx/client_temp \
      --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
      --user=nginx --group=nginx \
      --with-http_ssl_module --with-http_v2_module \
      --with-http_gzip_static_module --with-http_stub_status_module \
      --with-threads --with-file-aio \
      --without-http_autoindex_module --without-http_browser_module \
      --without-http_geo_module --without-http_map_module \
      --without-http_memcached_module --without-http_userid_module \
      --without-mail_pop3_module --without-mail_imap_module \
      --without-mail_smtp_module --without-http_split_clients_module \
      --without-http_uwsgi_module --without-http_scgi_module \
      --without-http_grpc_module && \
    make -j$(nproc) && make install && strip /usr/sbin/nginx && \
    rm -rf /tmp/nginx* && apk del .build-deps

# ======================================
# Giai đoạn 4: Runtime base tối giản
# ======================================
# Chỉ giữ runtime deps; tạo user non-root; chuẩn bị thư mục và quyền
FROM alpine:3.20@sha256:765942a4039992336de8dd5db680586e1a206607dd06170ff0a37267a9e01958 AS custom-runtime-base
RUN apk add --no-cache pcre2 zlib openssl tzdata && \
    addgroup -g 101 -S nginx && \
    adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx && \
    mkdir -p /var/cache/nginx /var/log/nginx /etc/nginx/conf.d /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx /var/log/nginx /usr/share/nginx/html
COPY --from=nginx-builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=nginx-builder /etc/nginx /etc/nginx

# ======================================
# Giai đoạn 5: Runtime cuối
# ======================================
FROM custom-runtime-base AS runtime
LABEL org.opencontainers.image.title="SvnFrs-Dockerfile_Contest_2025" \
      org.opencontainers.image.description="SPA production trên Alpine với Nginx tự biên dịch, tối ưu và bảo mật" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.created="2025-10-28" \
      org.opencontainers.image.base.name="alpine:3.20"

# Ứng dụng tĩnh đã build
COPY --from=builder --chown=nginx:nginx /app/dist /usr/share/nginx/html

# Cấu hình Nginx tối thiểu (inline) bật gzip_static để phục vụ file .gz
RUN echo 'server { \
  listen 3000; server_name localhost; \
  root /usr/share/nginx/html; index index.html; \
  gzip_static on; gzip_vary on; \
  location / { try_files $uri $uri/ /index.html; expires 1y; add_header Cache-Control "public, immutable"; } \
  location = /index.html { expires -1; add_header Cache-Control "no-cache"; } \
  add_header X-Frame-Options "SAMEORIGIN" always; \
  add_header X-Content-Type-Options "nosniff" always; \
  add_header X-XSS-Protection "1; mode=block" always; \
}' > /etc/nginx/conf.d/default.conf && \
echo 'worker_processes auto; error_log /var/log/nginx/error.log warn; pid /var/run/nginx.pid; \
events { worker_connections 1024; } http { include /etc/nginx/mime.types; default_type application/octet-stream; sendfile on; keepalive_timeout 65; include /etc/nginx/conf.d/*.conf; }' > /etc/nginx/nginx.conf && \
echo 'types { \
  text/html html htm shtml; text/css css; text/javascript js; application/json json; \
  image/svg+xml svg svgz; image/x-icon ico; image/png png; image/jpeg jpeg jpg; \
  font/woff2 woff2; application/wasm wasm; \
}' > /etc/nginx/mime.types

# Thư mục tạm/ngầm của Nginx + phân quyền trước khi chuyển USER
RUN mkdir -p /var/cache/nginx/client_temp \
             /var/cache/nginx/proxy_temp \
             /etc/nginx/fastcgi_temp \
             /etc/nginx/proxy_temp \
             /etc/nginx/client_body_temp \
             /etc/nginx/uwsgi_temp \
             /etc/nginx/scgi_temp && \
    chown -R nginx:nginx /var/cache/nginx \
                         /var/log/nginx \
                         /usr/share/nginx/html \
                         /etc/nginx/fastcgi_temp \
                         /etc/nginx/proxy_temp \
                         /etc/nginx/client_body_temp \
                         /etc/nginx/uwsgi_temp \
                         /etc/nginx/scgi_temp && \
    chmod -R 755 /var/cache/nginx \
                 /etc/nginx/fastcgi_temp \
                 /etc/nginx/proxy_temp \
                 /etc/nginx/client_body_temp \
                 /etc/nginx/uwsgi_temp \
                 /etc/nginx/scgi_temp && \
    touch /var/run/nginx.pid && \
    chown nginx:nginx /var/run/nginx.pid

# Healthcheck rẻ: HTTP 200 ở cổng 3000
RUN apk add --no-cache wget
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/ || exit 1

EXPOSE 3000
STOPSIGNAL SIGQUIT
USER nginx
CMD ["nginx", "-g", "daemon off;"]
```
# Top Dockerfile PYTHON
## Dockerfile TOP 1 (PYTHON)
```
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim@sha256:6b8ac7bb76766ffe9f6cc20f56789755d539e8d0e605d8983131227c5c8b87a1 AS builder
ENV UV_LINK_MODE=copy

ARG TARGETARCH
# Copy shared libraries đủ để chạy ứng dụng trong môi trường distroless
# Kiểm tra shared libraries cần thiết với lệnh:
# ldd $(which python3) và các thư viện khác trong virtual environment sau khi cài đặt các package cần thiết (kiểm tra trước khi build)
# Mỗi kiến trúc sẽ đặt thư viện trong các thư mục khác nhau, ví dụ: /lib/x86_64-linux-gnu/ cho amd64, /lib/aarch64-linux-gnu/ cho arm64
# do đó cần xác định kiến trúc và copy từ thư mục tương ứng. 
# TARGETARCH là built-in arg của docker buildx, tự động nhận giá trị (amd64 hoặc arm64) khi build multi-arch
# Shared libraries copy ở lệnh phía dưới là chưa đủ để chạy ứng dụng, tuy nhiên gcr.io/distroless/base-debian12 (image sử dụng làm base image cho runtime tại runtime state) đã có sẵn một số shared libraries nên chỉ cần copy những thư viện còn thiếu.
# gcr.io/distroless/base-debian12:nonroot không có shell, kiểm tra shared libraries bằng cách sử dụng gcr.io/distroless/base-debian12:debug
# gcr.io/distroless/base-debian12:debug tương tự gcr.io/distroless/base-debian12:nonroot nhưng có thêm shell để phục vụ debug.
RUN if [ "$TARGETARCH" = "amd64" ]; then \
        LIBARCH="x86_64"; \
    elif [ "$TARGETARCH" = "arm64" ]; then \
        LIBARCH="aarch64"; \
    else \
        LIBARCH="unknown"; \
    fi && \
    mkdir -p /lib/multi-arch && \
    cp /lib/${LIBARCH}-linux-gnu/libc.so.6 /lib/multi-arch/ && \
    cp /lib/${LIBARCH}-linux-gnu/libm.so.6 /lib/multi-arch/ && \
    cp /lib/${LIBARCH}-linux-gnu/libz.so.1 /lib/multi-arch/ && \
    cp /lib/${LIBARCH}-linux-gnu/libgcc_s.so.1 /lib/multi-arch/

WORKDIR /build

# Sử dụng cache để tăng tốc độ build
# Cài đặt dependencies trong uv virtual environment
# Sử dụng mount type=bind để bind các file uv.lock và pyproject.toml từ host vào container mount thay vì copy.
# --frozen để đảm bảo chỉ cài đặt đúng phiên bản dependencies trong uv.lock, không update uv.lock
# --no-install-project để không cài đặt project hiện tại (chỉ cài đặt dependencies)
# --no-dev để không cài đặt dev dependencies
# --no-editable để không cài đặt editable mode
# starlette 0.46.2 dính CVE-2025-62727 CVE-2025-54121, nâng cấp để vá lỗi bảo mật (do thay đổi pyproject.toml và file uv.lock sẽ vi phạm nội quy nên chạy lệnh install riêng)

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev --no-editable && \
    uv pip install "starlette==0.49.1" --no-deps

# Sử dụng distroless làm base image cho runtime để đảm bảo tính bảo mật và tối ưu kích thước image
# Chọn distroless thay vì alpine vì alpine sử dụng musl libc, trong khi python và nhiều thư viện phổ biến trong python được biên dịch với glibc, dẫn đến các vấn đề tương thích.
# distroless giúp ứng dụng chạy ổn định hơn và cũng rất nhẹ.
# cc-debian12 có nhiều shared libraries cần thiết cho python và các package phổ biến hơn so với base-debian12
# tuy nhiên khi đã kiểm tra kỹ các shared libraries cần thiết (với lệnh ldd) và copy đầy đủ từ builder stage thì base-debian12 sẽ giúp tối ưu kích thước image hơn mà vẫn đảm bảo ứng dụng chạy ổn định.
FROM gcr.io/distroless/base-debian12:nonroot@sha256:10136f394cbc891efa9f20974a48843f21a6b3cbde55b1778582195d6726fa85 AS runtime

LABEL maintainer="Thanh Nguyen The"
LABEL maintainer.email="thanhnt.devops@gmail.com"
LABEL maintainer.company="VIETNAM NATIONAL CYBER SECURITY TECHNOLOGY CORPORATION"
LABEL maintainer.youtube="DevOps Mentor"
LABEL image.description="Secure, minimal Python app using UV and Distroless"

WORKDIR /app

# Copy các thư viện và python từ builder stage
COPY --from=builder /lib/multi-arch/ /lib/multi-arch/
COPY --from=builder /usr/local/lib/libpython3.13.so.1.0 /usr/local/lib/libpython3.13.so.1.0
COPY --from=builder /usr/local/lib/python3.13/ /usr/local/lib/python3.13/
COPY --from=builder /usr/local/bin/python /usr/local/bin/python3
# Copy virtual environment từ builder
COPY --from=builder --chown=nonroot:nonroot /build/.venv/ /app/.venv/

# Copy source code - chỉ copy những gì cần thiết
COPY --chown=nonroot:nonroot src/ ./src/

# Thiết lập environment variables
# Do runtime limit là 1 vCPU, 512MB RAM nên thiết lập WORKERS=2 thay vì 3 (nguy cơ OOM). Công thức worker = (2 x số lượng vCPU + 1) chỉ áp dụng trong trường hợp > 1GB RAM
# LD_LIBRARY_PATH để hệ thống có thể tìm thấy các shared libraries cần thiết tại thư mục mới thay vì thư mục mặc đinh (/lib/x86_64-linux-gnu hoặc /lib/aarch64-linux-gnu)
# shared libraries không có trong /lib/multi-arch sẽ tiếp tục được load từ thư mục mặc định của hệ thống
ENV PATH="/app/.venv/bin/:$PATH" \
    PYTHONPATH="/app/src/" \
    LANG=C.UTF-8 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONHASHSEED=random \
    HOST=0.0.0.0 \
    PORT=8080 \
    WORKERS=2 \
    LOGGING__LEVEL=INFO \
    LOGGING__FORMAT=PLAIN \
    COFFEE_API__HOST="https://api.sampleapis.com/coffee/" \
    APP_VERSION=0.1.0 \
    GIT_COMMIT_SHA=sha \
    LD_LIBRARY_PATH=/lib/multi-arch

# nonroot user mặc định đã được sử dụng trong distroless base-debian12:nonroot nên không cần thiết phải thêm lệnh phía dưới
# USER nonroot:nonroot

# Expose port mặc định
EXPOSE 8080

# Command để chạy ứng dụng
ENTRYPOINT ["python", "src/python_service_template/app.py"]
```
## Dockerfile TOP 2 (PYTHON)
```
# syntax=docker/dockerfile:1.7

# =============================================================================
# DOCKERFILE ULTRA OPTIMIZED + SECURITY PATCHED
# Mục tiêu: Nhẹ (<110MB) + Bảo mật cao (0 CVEs)
# =============================================================================

# -----------------------------------------------------------------------------
# Stage 1: Dependencies Builder
# -----------------------------------------------------------------------------
FROM python:3.13-alpine@sha256:e5fa639e49b85986c4481e28faa2564b45aa8021413f31026c3856e5911618b1 AS deps

ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache --virtual .build-deps \
      build-base \
      python3-dev \
      cargo

# Install dependencies với PATCHED versions để fix CVEs
# Note: FastAPI 0.116+ required for starlette 0.49.1+ support
RUN --mount=type=cache,target=/root/.cache/pip \
    python -m pip install --no-cache-dir --prefix=/install \
         "aiohttp>=3.12.14,<4.0.0" \
         "asgi-correlation-id>=4.3.4,<5.0.0" \
         "fastapi>=0.116.0" \
         "prometheus-fastapi-instrumentator>=7.0.0,<8.0.0" \
         "pydantic>=2.11.0,<3.0.0" \
         "pydantic-settings>=2.9.1,<3.0.0" \
         "structlog>=25.3.0,<26.0.0" \
         "uvloop>=0.21.0,<0.22.0" \
         "uvicorn[standard]>=0.30.0,<0.31.0"

# ULTRA AGGRESSIVE optimization
RUN apk add --no-cache binutils \
 # Strip ALL .so files aggressively
 && find /install -type f \( -name '*.so*' -o -name '*.a' \) -exec strip --strip-all {} + 2>/dev/null || true \
 # Remove all bytecode
 && find /install \( -type d -name __pycache__ -o -type f -name '*.py[co]' \) -delete 2>/dev/null || true \
 # Remove test/doc/examples
 && find /install -type d \( -name tests -o -name testing -o -name test -o -name doc -o -name docs -o -name example -o -name examples \) -prune -exec rm -rf {} + 2>/dev/null || true \
 # Minimize .dist-info
 && find /install -name '*.dist-info' -type d -exec sh -c 'cd "$1" && find . -type f ! -name "METADATA" ! -name "top_level.txt" ! -name "RECORD" -delete' _ {} \; 2>/dev/null || true \
 # Remove typing stubs, headers, C files
 && find /install -type f \( -name '*.pyi' -o -name '*.c' -o -name '*.h' -o -name '*.cpp' -o -name '*.cc' \) -delete \
 # Remove license files
 && find /install -type f \( -name 'LICENSE*' -o -name 'COPYING*' -o -name 'NOTICE*' -o -name 'AUTHORS*' -o -name 'CHANGELOG*' -o -name 'README*' \) -delete 2>/dev/null || true \
 && apk del binutils .build-deps

# -----------------------------------------------------------------------------
# Stage 2: Runtime (ULTRA MINIMAL + SECURE)
# -----------------------------------------------------------------------------
FROM python:3.13-alpine@sha256:e5fa639e49b85986c4481e28faa2564b45aa8021413f31026c3856e5911618b1 AS runtime

LABEL org.opencontainers.image.title="Python Service Template" \
      org.opencontainers.image.description="Production-ready FastAPI service - Optimized & Secured" \
      org.opencontainers.image.version="0.1.0" \
      org.opencontainers.image.authors="newnol <contact@newnol.io.vn>" \
      maintainer="newnol" \
      security.scan="trivy-passed"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    HOST=0.0.0.0 \
    PORT=5000 \
    WORKERS=1 \
    PYTHONPATH=/app/src \
    TZ=UTC

# Install ONLY wget for healthcheck
RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache wget

WORKDIR /app

# Create non-root user
RUN addgroup -g 10001 -S app \
 && adduser -u 10001 -S -G app -h /app -s /sbin/nologin app

# Copy dependencies
COPY --from=deps --chown=app:app /install /usr/local

# Copy source (minimal)
COPY --chown=app:app src/ ./src/

# Permissions
RUN chmod -R 550 /app

# EXTREME Python stdlib cleanup
RUN rm -rf \
    /usr/local/lib/python3.13/ensurepip \
    /usr/local/lib/python3.13/site-packages/pip* \
    /usr/local/lib/python3.13/site-packages/setuptools* \
    /usr/local/lib/python3.13/site-packages/wheel* \
    /usr/local/lib/python3.13/distutils \
    /usr/local/lib/python3.13/lib2to3 \
    /usr/local/lib/python3.13/idlelib \
    /usr/local/lib/python3.13/tkinter \
    /usr/local/lib/python3.13/turtledemo \
    /usr/local/lib/python3.13/test \
    /usr/local/lib/python3.13/unittest/test \
    /usr/local/bin/pip* \
    /usr/local/bin/2to3* \
    /usr/local/bin/idle* \
    2>/dev/null || true

# Clean up more unused stdlib modules
RUN cd /usr/local/lib/python3.13 && rm -rf \
    turtle.py \
    pydoc_data \
    2>/dev/null || true

USER app

EXPOSE 5000

# Heathcheck 
HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=2 \
    CMD wget --no-verbose --tries=1 -O /dev/null http://127.0.0.1:${PORT}/health/ || exit 1

STOPSIGNAL SIGTERM

CMD ["python", "src/python_service_template/app.py"]
```
## Dockerfile TOP (PYTHON)
```
# syntax=docker/dockerfile:1.7
#
##############################
# builder
##############################
FROM python:3.13-slim AS builder

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_ONLY_BINARY=:all: \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Pre-cache manifests
COPY pyproject.toml README.md LICENSE* ./

# Fix HIGH vulnerable issue: CVE-2025-62727 by upgrading starlette and fastapi.
RUN --mount=type=cache,target=/root/.cache/pip python - <<'PY'
import tomllib, pathlib, re
def parse_req(s:str):
    m = re.match(r'^\s*([A-Za-z0-9_.-]+)(\[[^\]]+\])?\s*(.*)$', s)
    if m:
        name, extras, rest = m.group(1), (m.group(2) or ''), (m.group(3) or '')
        return name, extras, rest
    name = re.split(r'[><=~!; ]', s, 1)[0]
    return name, '', s[len(name):]

data = tomllib.loads(pathlib.Path('pyproject.toml').read_text())
deps = data.get('project', {}).get('dependencies', [])
safe = []
present = set()
for d in deps:
    name, extras, rest = parse_req(d)
    norm = name.lower().replace('_','-')
    if norm == 'fastapi':
        safe.append(f'fastapi{extras}>=0.118,<0.121')
    else:
        safe.append(d)
    present.add(norm)
if 'starlette' not in present:
    safe.append('starlette>=0.49.1,<0.50')
pathlib.Path('/requirements.safe.txt').write_text('\n'.join(safe) + '\n')
print('Resolved safe deps:', *safe, sep='\n- ')
PY

# Wheel ALL dependencies from the safe list
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --wheel-dir /wheels -r /requirements.safe.txt

# Build wheel of the project itself
COPY src/ ./src/
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --wheel-dir /wheels .


##############################
# runtime
##############################
FROM python:3.13-slim AS runtime

ARG VERSION=0.1.0
ARG VCS_REF=sha
ARG BUILD_DATE

LABEL org.opencontainers.image.title="python-service-template" \
      org.opencontainers.image.description="Dockerfile contest build" \
      org.opencontainers.image.version=$VERSION \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.licenses="Apache-2.0"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONOPTIMIZE=2 \
    HOST=0.0.0.0 \
    PORT=5000 \
    WORKERS=1 \
    APP_VERSION=$VERSION \
    GIT_COMMIT_SHA=$VCS_REF

# Non-root
RUN useradd --create-home --uid 10001 --shell /usr/sbin/nologin appuser
WORKDIR /home/appuser

# Install offline: install ALL safe deps, then the app wheel with --no-deps
COPY --from=builder /wheels /wheels
COPY --from=builder /requirements.safe.txt /requirements.safe.txt
RUN pip install --no-index --find-links=/wheels -r /requirements.safe.txt \
    && pip install --no-index --find-links=/wheels --no-deps \
       python-service-template --no-compile \
    && rm -rf /wheels /requirements.safe.txt

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=2s --start-period=10s --retries=3 \
  CMD python -c "import sys, http.client; c=http.client.HTTPConnection('127.0.0.1', int(__import__('os').environ.get('PORT','5000')), timeout=1); c.request('GET','/health'); r=c.getresponse(); sys.exit(0 if r.status==200 else 1)" || exit 1

USER 10001:10001

CMD ["python", "-m", "python_service_template.app"]
```
## Dockerfile TOP (PYTHON) 2
```
# syntax=docker/dockerfile:1.19

FROM python:3.13-alpine3.22 AS deps

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-compile uv==0.9.2

WORKDIR /app

COPY pyproject.toml uv.lock ./

RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project \
    && uv pip install starlette==0.50.0

FROM python:3.13-alpine3.22 AS final

RUN pip install -U pip

RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache \
    curl=8.14.1-r2 \
    tini=0.19.0-r3

ARG VERSION="0.1.0"
ARG BUILD_DATE="2025-11-10T00:00:00Z"
ARG REVISION="unknown"
ARG GIT_COMMIT_SHA="unknown"

LABEL org.opencontainers.image.title="python-service-template" \
      org.opencontainers.image.description="A batteries-included template for building robust, production-ready Python backend services with FastAPI" \
      org.opencontainers.image.authors="Khiem Doan" \
      org.opencontainers.image.version=$VERSION \
      org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$REVISION \
      org.opencontainers.image.source="https://github.com/khiemdoan/" \
      org.opencontainers.image.licenses="MIT"

ENV USER=nonroot \
    GROUP=nonroot \
    UID=14406 \
    GID=14406

RUN addgroup -g "$GID" "$GROUP" \
    && adduser -D -u "$UID" -G "$GROUP" "$USER"

USER $USER

WORKDIR /app

COPY --chown=$USER:$GROUP src/ src/

COPY --from=deps --chown=$USER:$GROUP /app/.venv /app/.venv

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH="/app/src" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    HOST=0.0.0.0 \
    PORT=3000 \
    WORKERS=1 \
    LOGGING__LEVEL=INFO \
    LOGGING__FORMAT=PLAIN \
    COFFEE_API__HOST=https://api.sampleapis.com/coffee/ \
    APP_VERSION=$VERSION \
    GIT_COMMIT_SHA=$GIT_COMMIT_SHA

EXPOSE $PORT

HEALTHCHECK --timeout=1s \
    CMD curl -f "http://localhost:${PORT}/health/" | grep '"heartbeat":"HEALTHY"' || exit 1

ENTRYPOINT ["/sbin/tini", "--"]

CMD ["python", "src/python_service_template/app.py"]
```
# Top Dockerfile JAVA
## Dockerfile TOP 1 (JAVA)
```
# Build stage
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app

# Copy gradle wrapper and properties first for better caching
COPY gradlew gradlew.bat build.gradle ./
COPY gradle/ gradle/

# Download Gradle distribution (cached)
RUN --mount=type=cache,target=/root/.gradle ./gradlew --version

# Copy source code
COPY src/ src/

# Build the application
RUN --mount=type=cache,target=/root/.gradle ./gradlew --no-daemon clean bootJar \
    -Dspring-framework.version=6.2.11 \
    -Dtomcat.version=10.1.47

# Extract the application dependencies
RUN jar xf build/libs/spring-boot-template.jar

# Analyze the dependencies contained into the fat jar
RUN jdeps --ignore-missing-deps -q  \
  --recursive  \
  --multi-release 21  \
  --print-module-deps  \
  --class-path 'BOOT-INF/lib/*'  \
  build/libs/spring-boot-template.jar > deps.info

# Create the custom JRE
RUN jlink \
  --verbose \
  --add-modules $(cat deps.info) \
  --compress zip-9 \
  --no-header-files \
  --no-man-pages \
  --output /custom_jre

# Healthcheck stage
FROM busybox:1.36.0-musl AS healthcheck

# Runtime stage
FROM gcr.io/distroless/base-debian12
ENV JAVA_HOME=/opt/java/openjdk
ENV PATH="$JAVA_HOME/bin:$PATH"
COPY --from=build /custom_jre $JAVA_HOME

# Copy wget for healthcheck
COPY --from=healthcheck /bin/wget /usr/bin/wget

WORKDIR /app

# Copy application insights config
COPY lib/applicationinsights.json ./

# Copy the built JAR
COPY --from=build /app/build/libs/spring-boot-template.jar /app.jar

# Add labels
LABEL org.opencontainers.image.source="https://github.com/hmcts/spring-boot-template" \
  org.opencontainers.image.version="0.0.1" \
  org.opencontainers.image.licenses="MIT"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["/usr/bin/wget", "--quiet", "--output-document=/dev/null", "http://localhost:8080/health"]

CMD ["java", "-jar", "/app.jar"]
```
## Dockerfile TOP 2 (JAVA)
```
# Stage 1 — Build application using Gradle
FROM gradle:8.14.3-jdk21-alpine AS builder

WORKDIR /app

# Caching wrapper and build configuration before build 
COPY gradlew ./
COPY gradle gradle
COPY build.gradle build.gradle

# Caching gradle/download
RUN ./gradlew --no-daemon help

# Copy src code
COPY src/main src/main

# Check dependency need to update
RUN ./gradlew --no-daemon dependencyUpdates -Drevision=release

# Auto update for auto vulnerability fixing 
RUN REPORT_FILE="build/dependencyUpdates/report.txt" && \
    echo "=== Parsing $REPORT_FILE ===" && \
    \
    PLUGINS_TO_UPGRADE=${PLUGINS_TO_UPGRADE:-"org.springframework.boot org.sonarqube com.github.ben-manes.versions uk.gov.hmcts.java"} && \
    EXTS_TO_UPGRADE=${EXTS_TO_UPGRADE:-"org.apache.logging.log4j ch.qos.logback"} && \
    DEPENDENCIES_FORCE_UPDATE=${DEPENDENCIES_FORCE_UPDATE:-"org.apache.commons:commons-lang3:3.19.0"} && \
    \
    escape_sed() { printf '%s\n' "$1" | sed 's/[.[\*^$/&]/\\&/g'; } && \
    \
    # --- Plugin updates ---
    for plugin in $PLUGINS_TO_UPGRADE; do \
    LINE=$(grep -A1 "$plugin" "$REPORT_FILE" | grep '\[' | head -1 || true); \
    OLD_VERSION=$(echo "$LINE" | sed -E 's/.*\[(.*) -> .*\].*/\1/' || true); \
    NEW_VERSION=$(echo "$LINE" | sed -E 's/.*\[.* -> (.*)\].*/\1/' || true); \
    if [ -n "$NEW_VERSION" ] && [ "$NEW_VERSION" != "$OLD_VERSION" ]; then \
    echo "===== Upgrading plugin $plugin: $OLD_VERSION → $NEW_VERSION"; \
    ESC_OLD=$(escape_sed "$OLD_VERSION"); \
    ESC_NEW=$(escape_sed "$NEW_VERSION"); \
    sed -i "s#id '$plugin' version '$ESC_OLD'#id '$plugin' version '$ESC_NEW'#g" build.gradle; \
    fi; \
    done && \
    \
    # --- ext{} version updates ---
    for prefix in $EXTS_TO_UPGRADE; do \
    LINE=$(grep -A1 "$prefix" "$REPORT_FILE" | grep '\[' | head -1 || true); \
    OLD_VERSION=$(echo "$LINE" | sed -E 's/.*\[(.*) -> .*\].*/\1/' || true); \
    NEW_VERSION=$(echo "$LINE" | sed -E 's/.*\[.* -> (.*)\].*/\1/' || true); \
    if [ -n "$NEW_VERSION" ] && [ "$NEW_VERSION" != "$OLD_VERSION" ]; then \
    echo "===== Upgrading prefix $prefix: $OLD_VERSION → $NEW_VERSION"; \
    ESC_OLD=$(escape_sed "$OLD_VERSION"); \
    ESC_NEW=$(escape_sed "$NEW_VERSION"); \
    sed -i "s/\"$ESC_OLD\"/\"$ESC_NEW\"/g" build.gradle; \
    fi; \
    done && \
    \
    # --- Force dependency updates with explicit GAV ---
    for dep in $DEPENDENCIES_FORCE_UPDATE; do \
    GROUP=$(echo "$dep" | cut -d':' -f1); \
    NAME=$(echo "$dep" | cut -d':' -f2); \
    NEW_VERSION=$(echo "$dep" | cut -d':' -f3); \
    if [ -z "$GROUP" ] || [ -z "$NAME" ] || [ -z "$NEW_VERSION" ]; then \
    echo "======  Invalid DEPENDENCIES_FORCE_UPDATE format for $dep, expected group:name:version"; \
    continue; \
    fi; \
    echo "====== Forcing dependency update: $GROUP:$NAME → $NEW_VERSION"; \
    ESC_GROUP=$(escape_sed "$GROUP"); \
    ESC_NAME=$(escape_sed "$NAME"); \
    ESC_NEW=$(escape_sed "$NEW_VERSION"); \
    if grep -q "$ESC_GROUP" build.gradle | grep -q "$ESC_NAME"; then \
    # Replace existing dependency version
    sed -i "s#group: '$ESC_GROUP', name: '$ESC_NAME', version: '[^']*'#group: '$ESC_GROUP', name: '$ESC_NAME', version: '$ESC_NEW'#g" build.gradle; \
    else \
    # Insert new dependency inside dependencies { }
    echo "====== Adding new dependency $GROUP:$NAME:$NEW_VERSION"; \
    sed -i "/dependencies\s*{/a\    implementation group: '$GROUP', name: '$NAME', version: '$NEW_VERSION'" build.gradle; \
    fi; \
    done && \
    \
    echo "Version upgrade complete!" && \
    cat build.gradle

# Build the application JAR after dependency check
RUN ./gradlew --no-daemon bootJar

# Generate java healthcheck class
RUN mkdir -p /app/health && cat > /app/health/HealthCheck.java <<'EOF'
import java.net.HttpURLConnection;
import java.net.URL;
import java.time.Instant;

public class HealthCheck {
    public static void main(String[] args) {
        String healthUrl = "http://localhost:8080/health";
        try {
            URL url = new URL(healthUrl);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setConnectTimeout(2000);
            conn.setReadTimeout(2000);
            conn.setRequestMethod("GET");

            int code = conn.getResponseCode();
            if (code == 200) {
                System.out.println(Instant.now() + "Healthcheck OK (" + code + ")");
                System.exit(0);
            } else {
                System.err.println(Instant.now() + "Healthcheck failed (" + code + ")");
                System.exit(1);
            }
        } catch (Exception e) {
            System.err.println(Instant.now() + " Healthcheck error: " + e.getMessage());
            System.exit(1);
        }
    }
}
EOF

# Compile HealthCheck.java file
RUN javac /app/health/HealthCheck.java

# Stage 2 — Runtime image (auto-updated base)
FROM hmctspublic.azurecr.io/base/java:21-distroless

WORKDIR /app

# Copy compiled app
COPY --from=builder /app/build/libs/*.jar app.jar

# Copy complied healthcheck class
COPY --from=builder /app/health/HealthCheck.class /app/HealthCheck.class

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
    CMD ["java", "HealthCheck"]
```
## Dockerfile TOP (JAVA)
```
# Dockerfile.txt
# === Alpine build optimized for Dockerfile Contest 2025 ===
# Focus: security, lightweight, clear, maintainable

# Build-time arguments
ARG BUILD_JDK_IMAGE=eclipse-temurin:21-jdk-alpine-3.22
ARG RUNTIME_IMAGE=eclipse-temurin:21-jre-alpine-3.22

# ---------- Stage: builder ----------
FROM ${BUILD_JDK_IMAGE} AS builder

# Set non-interactive environment & reproducible timezone
ENV TZ=UTC \
  LANG=C.UTF-8 \
  LC_ALL=C.UTF-8 \
  GRADLE_USER_HOME=/cache/.gradle

WORKDIR /workspace

# Copy Gradle wrapper and descriptors
COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle ./

RUN chmod +x ./gradlew

# Resolve dependencies using BuildKit cache mount
RUN --mount=type=cache,target=/cache/.gradle \
  ./gradlew --no-daemon dependencies || true

# Copy application source
COPY src/ src/

RUN --mount=type=cache,target=/cache/.gradle \
  ./gradlew --no-daemon clean bootJar -x test \
  -Dspring-framework.version=6.2.11 \
  -Dcommons-lang3.version=3.18.0 \
  -Dtomcat.version=10.1.47

# ---------- Stage: runtime ----------
FROM ${RUNTIME_IMAGE} AS runtime

ARG VERSION
ARG VCS_REF
ARG BUILD_DATE
ARG LICENSE="MIT License"
ARG SOURCE="contest-submission"

# OCI Labels (metadata)
LABEL org.opencontainers.image.title="spring-boot-template" \
  org.opencontainers.image.description="Spring Boot Java application built for Dockerfile Contest 2025" \
  org.opencontainers.image.url="${SOURCE}" \
  org.opencontainers.image.source="${SOURCE}" \
  org.opencontainers.image.version="${VERSION}" \
  org.opencontainers.image.revision="${VCS_REF}" \
  org.opencontainers.image.licenses="${LICENSE}" \
  org.opencontainers.image.created="${BUILD_DATE}" \
  org.opencontainers.image.authors="Dung Cao"

# Create non-root user
RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

# Copy application jar
COPY lib/applicationinsights.json applicationinsights.json
COPY --from=builder --chown=app:app /workspace/build/libs/*.jar app.jar

# Install curl for healthcheck
RUN apk add --no-cache curl \
  && rm -rf /var/cache/apk/*

# Expose HTTP port
EXPOSE 8080

# Healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -fsS http://127.0.0.1:8080/health || exit 1

# JVM optimization
ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -Djava.security.egd=file:/dev/./urandom \
  -Dserver.shutdown=graceful \
  -Dspring.lifecycle.timeout-per-shutdown-phase=10s \
  -Dfile.encoding=UTF-8 \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+UseG1GC \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp \
  "

# Graceful termination signal
STOPSIGNAL SIGTERM

# Switch to non-root user
USER app

# --- Entry point ---
ENTRYPOINT ["sh", "-c", "exec java ${JAVA_OPTS} -jar /app/app.jar"]
```
## Dockerfile TOP - tham khảo (JAVA)
```
# syntax=docker/dockerfile:1.7

# ============================================================================
# STAGE 1: Application builder with optimized caching
# ============================================================================
FROM eclipse-temurin:21-jdk-alpine@sha256:89517925fa675c6c4b770bee7c44d38a7763212741b0d6fca5a5103caab21a97 AS builder

# Install build dependencies (minimal)
RUN apk add --no-cache binutils && \
  rm -rf /var/cache/apk/*

WORKDIR /build

# Copy Gradle wrapper and dependency definition files first
# This layer will be cached until these files change
COPY gradle/ gradle/
COPY gradlew build.gradle ./

# Download dependencies with BuildKit cache mount for faster subsequent builds
RUN --mount=type=cache,id=gradle-cache,target=/root/.gradle,sharing=locked \
  chmod +x gradlew && \
  ./gradlew dependencies --no-daemon --parallel --console=plain

# Copy only production source code (exclude tests, docs, etc.)
COPY src/main/ src/main/

# Build optimized JAR with cache mount
RUN --mount=type=cache,id=gradle-cache,target=/root/.gradle,sharing=locked \
  ./gradlew bootJar --no-daemon --parallel --console=plain -x test && \
  mkdir -p /app && \
  mv build/libs/spring-boot-template.jar /app/app.jar

# Extract Spring Boot layers for optimal Docker layer caching
WORKDIR /app
RUN java -Djarmode=layertools -jar app.jar extract --destination /app/extracted

# Create minimal custom JRE with jlink (reduces size by >100MB)
# Only include Java modules actually needed by Spring Boot
RUN $JAVA_HOME/bin/jlink \
  --add-modules java.base,java.compiler,java.desktop,java.instrument,java.management,java.management.rmi,java.naming,java.net.http,java.prefs,java.rmi,java.scripting,java.security.jgss,java.security.sasl,java.sql,jdk.httpserver,jdk.jfr,jdk.unsupported \
  --strip-debug \
  --no-man-pages \
  --no-header-files \
  --compress=zip-9 \
  --output /jre-minimal

# ============================================================================
# STAGE 3: Minimal runtime image
# ============================================================================
FROM alpine:3.21@sha256:5405e8f36ce1878720f71217d664aa3dea32e5e5df11acbf07fc78ef5661465b

# Install only critical runtime dependencies
# ca-certificates: for HTTPS connections
# tini: proper init system for PID 1
# tzdata: timezone support
# curl: for healthcheck
RUN apk upgrade --no-cache && \
  apk add --no-cache \
  ca-certificates \
  tzdata \
  tini \
  curl && \
  rm -rf /var/cache/apk/* /tmp/*

# Create non-root user for security (CIS Docker Benchmark compliance)
RUN addgroup -g 1654 -S appgroup && \
  adduser -u 1654 -S appuser -G appgroup

# Copy minimal custom JRE from builder
COPY --from=builder --chown=1654:1654 /jre-minimal /opt/java

# Set up application directory with proper ownership
WORKDIR /app

# Copy Spring Boot layers in optimal order (least to most frequently changed)
# This maximizes Docker layer cache efficiency
COPY --from=builder --chown=1654:1654 /app/extracted/dependencies/ ./
COPY --from=builder --chown=1654:1654 /app/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=1654:1654 /app/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=1654:1654 /app/extracted/application/ ./

# Switch to non-root user (security best practice)
USER 1654:1654

# Set JAVA_HOME and PATH
ENV JAVA_HOME=/opt/java \
  PATH="/opt/java/bin:${PATH}"

# Optimal JVM flags for containerized Spring Boot applications
# - UseContainerSupport: respect container memory limits
# - MaxRAMPercentage: use max 75% of container memory for heap
# - UseG1GC: best GC for containers with predictable pause times
# - UseStringDeduplication: reduce memory footprint
# - ExitOnOutOfMemoryError: fail fast on OOM
# - TieredCompilation with level 1: faster startup, good for short-lived containers
ENV JAVA_TOOL_OPTIONS="-XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -XX:+UseStringDeduplication \
  -XX:+ParallelRefProcEnabled \
  -XX:+DisableExplicitGC \
  -XX:+ExitOnOutOfMemoryError \
  -Djava.security.egd=file:/dev/./urandom \
  -Djava.awt.headless=true"

# Application server port
EXPOSE 8080

# Comprehensive OCI labels for traceability and compliance
LABEL org.opencontainers.image.title="Spring Boot Template" \
  org.opencontainers.image.description="HMCTS Spring Boot Template - Optimized for Contest 2025" \
  org.opencontainers.image.vendor="HMCTS Reform Programme" \
  org.opencontainers.image.authors="HMCTS <hmcts@justice.gov.uk>" \
  org.opencontainers.image.source="https://github.com/hmcts/spring-boot-template" \
  org.opencontainers.image.version="0.0.1" \
  org.opencontainers.image.revision="contest-2025" \
  org.opencontainers.image.licenses="MIT" \
  org.opencontainers.image.base.name="docker.io/library/alpine:3.21" \
  org.opencontainers.image.base.digest="sha256:5405e8f36ce1878720f71217d664aa3dea32e5e5df11acbf07fc78ef5661465b" \
  maintainer="HMCTS Reform Team" \
  com.hmcts.app.name="spring-boot-template" \
  com.hmcts.build.date="2025-10-27"

# Health check using Spring Boot Actuator /health endpoint
# Using curl for lightweight health checks
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Use tini as init system for proper signal handling
# Ensures graceful shutdown and zombie process reaping
ENTRYPOINT ["/sbin/tini", "--"]

# Run Spring Boot application
# Using exec form to ensure proper signal propagation
CMD ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```