# Copyright 2015 The Kubernetes Authors. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG BASE_IMAGE

# Stage 1: Add Lua dependencies to the base image
FROM ${BASE_IMAGE} AS lua-deps

USER root

# Install LuaRocks for proper package management
RUN apk update \
  && apk add --no-cache \
    build-base \
    unzip \
    wget \
    git \
    openssl-dev \
    pcre-dev \
    zlib-dev \
    luarocks5.1

# Install Lua dependencies via LuaRocks (proper package management)
RUN luarocks-5.1 install lua-resty-lrucache \
  && luarocks-5.1 install lua-resty-lock \
  && luarocks-5.1 install lua-resty-mlcache

# Note: Verification requires OpenResty context, so we'll verify at runtime

# Stage 2: Build the final image
FROM ${BASE_IMAGE}

ARG TARGETARCH
ARG VERSION
ARG COMMIT_SHA
ARG BUILD_ID=UNSET

LABEL org.opencontainers.image.title="NGINX Ingress Controller for Kubernetes with Arxignis"
LABEL org.opencontainers.image.documentation="https://kubernetes.github.io/ingress-nginx/"
LABEL org.opencontainers.image.source="https://github.com/arxignis/ingress-nginx"
LABEL org.opencontainers.image.vendor="Arxignis"
LABEL org.opencontainers.image.licenses="Apache-2.0"
LABEL org.opencontainers.image.version="${VERSION}"
LABEL org.opencontainers.image.revision="${COMMIT_SHA}"

LABEL org.opencontainers.image.description="Arxignis-enabled NGINX Ingress Controller for Kubernetes with full Lua dependencies"


LABEL build_id="${BUILD_ID}"

WORKDIR  /etc/nginx


# Install Lua dependencies for Arxignis
USER root
RUN apk update \
  && apk add --no-cache \
    build-base \
    unzip \
    wget \
    git \
    openssl-dev \
    pcre-dev \
    zlib-dev \
    luarocks5.1 \
    diffutils \

  && luarocks-5.1 install lua-resty-mlcache \
  && luarocks-5.1 install lua-resty-lrucache \
  && luarocks-5.1 install lua-resty-lock \
  && apk del build-base unzip wget git openssl-dev pcre-dev zlib-dev luarocks5.1 \
  && rm -rf /var/cache/apk/*

# Create minimal worker module for Arxignis
RUN mkdir -p /etc/nginx/lua/resty/arxignis \
  && cat > /etc/nginx/lua/resty/arxignis/worker.lua << 'EOF'
local _M = {}

function _M.start_flush_timers(config)
    ngx.log(ngx.INFO, "Arxignis worker initialized with API URL: " .. (config.ARXIGNIS_API_URL or "not set"))
    -- Placeholder for background tasks
    -- In a full implementation, this would start timers for database updates
end

return _M
EOF

# Copy existing ingress-nginx Lua modules and configuration
COPY --chown=www-data:www-data rootfs/etc /etc


# Note: Lua modules will be verified at runtime in OpenResty context
# Standard lua5.1 cannot test resty.* modules as they require LuaJIT/OpenResty FFI
RUN echo "✅ Lua dependencies installed via LuaRocks - verification will happen at runtime"

COPY --chown=www-data:www-data bin/${TARGETARCH}/dbg /
COPY --chown=www-data:www-data bin/${TARGETARCH}/nginx-ingress-controller /
COPY --chown=www-data:www-data bin/${TARGETARCH}/wait-shutdown /

# Fix permission during the build to avoid issues at runtime
# with volumes (custom templates)
RUN bash -xeu -c ' \
  writeDirs=( \
    /etc/ingress-controller \
    /etc/ingress-controller/ssl \
    /etc/ingress-controller/auth \
    /etc/ingress-controller/geoip \
    /etc/ingress-controller/telemetry \
    /var/log \
    /var/log/nginx \
    /tmp/nginx \
  ); \
  for dir in "${writeDirs[@]}"; do \
    mkdir -p ${dir}; \
    chown -R www-data:www-data ${dir}; \
  done' \
  # LD_LIBRARY_PATH does not work so below is needed for  opentelemetry/other modules
  # Put libs of newer modules under `/modules_mount/<other>/lib` and add that path below
  # Could get complicated arch specific paths become a need
  && echo "/lib:/usr/lib:/usr/local/lib:/modules_mount/etc/nginx/modules/otel" > /etc/ld-musl-x86_64.path

RUN apk add --no-cache libcap \
  && setcap cap_net_bind_service=+ep /nginx-ingress-controller \
  && getcap /nginx-ingress-controller | grep -q 'cap_net_bind_service=ep' \
  && if [ -f /usr/local/openresty/nginx/sbin/nginx ]; then \
       setcap cap_net_bind_service=+ep /usr/local/openresty/nginx/sbin/nginx && \
       getcap /usr/local/openresty/nginx/sbin/nginx | grep -q 'cap_net_bind_service=ep'; \
     fi \
  && if [ -f /usr/bin/dumb-init ]; then \
       setcap cap_net_bind_service=+ep /usr/bin/dumb-init && \
       getcap /usr/bin/dumb-init | grep -q 'cap_net_bind_service=ep'; \
     fi \
  && apk del libcap \
  && if [ -x /usr/local/openresty/nginx/sbin/nginx ]; then ln -sf /usr/local/openresty/nginx/sbin/nginx /usr/bin/nginx; \
     elif [ -x /usr/local/nginx/sbin/nginx ]; then ln -sf /usr/local/nginx/sbin/nginx /usr/bin/nginx; \
     elif [ -x /sbin/nginx ]; then ln -sf /sbin/nginx /usr/bin/nginx; \
     else echo "nginx binary not found in expected locations" >&2; exit 1; fi

USER www-data

# Create symlinks to redirect nginx logs to stdout and stderr docker log collector
RUN  ln -sf /dev/stdout /var/log/nginx/access.log \
  && ln -sf /dev/stderr /var/log/nginx/error.log

# Lua modules will be verified when Arxignis is enabled in the controller

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/nginx-ingress-controller"]
