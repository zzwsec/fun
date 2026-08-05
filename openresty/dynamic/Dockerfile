# syntax=docker/dockerfile:1

# Dynamic modules layered on the OpenResty base image.

ARG RESTY_IMAGE_TAG="alpine:3.24.1"
ARG OPENRESTY_IMAGE="zzwsec/openresty:1.31.1-alpine"

FROM ${RESTY_IMAGE_TAG} AS modules-builder

ARG RESTY_VERSION="1.31.1.1"
ARG NGINX_VERSION="1.31.1"
ARG BUILD_JOBS=""

ARG GEOIP2_REPO="https://github.com/leev/ngx_http_geoip2_module.git"
ARG GEOIP2_REF="3.4"
ARG OTEL_REPO="https://github.com/nginx/nginx-otel.git"
ARG OTEL_REF="v0.1.2"
ARG CACHE_PURGE_REPO="https://github.com/nginx-modules/ngx_cache_purge.git"
ARG CACHE_PURGE_REF="3.0.2"
ARG FANCYINDEX_REPO="https://github.com/aperezdc/ngx-fancyindex.git"
ARG FANCYINDEX_REF="v0.6.0"

RUN apk add --no-cache \
        build-base c-ares-dev cmake curl git linux-headers libmaxminddb-dev \
        openssl-dev pcre2-dev zlib-dev \
    && curl -sSfL "https://openresty.org/download/openresty-${RESTY_VERSION}.tar.gz" | tar xzf - -C /tmp \
    && git clone --depth=1 --branch "${GEOIP2_REF}" "${GEOIP2_REPO}" /tmp/ngx_http_geoip2_module \
    && git clone --depth=1 --branch "${OTEL_REF}" "${OTEL_REPO}" /tmp/nginx-otel \
    && git clone --depth=1 --branch "${CACHE_PURGE_REF}" "${CACHE_PURGE_REPO}" /tmp/ngx_cache_purge \
    && git clone --depth=1 --branch "${FANCYINDEX_REF}" "${FANCYINDEX_REPO}" /tmp/ngx-fancyindex

WORKDIR /tmp/openresty-${RESTY_VERSION}/bundle/nginx-${NGINX_VERSION}
RUN ./configure \
        --with-compat \
        --with-stream \
        --add-dynamic-module=/tmp/ngx_http_geoip2_module \
        --add-dynamic-module=/tmp/ngx_cache_purge \
        --add-dynamic-module=/tmp/ngx-fancyindex \
    && MODULE_BUILD_JOBS="${BUILD_JOBS:-$(nproc)}" \
    && make -j"${MODULE_BUILD_JOBS}" modules

# https://github.com/nginx/nginx-otel/blob/main/config
# https://cmake.org/cmake/help/latest/variable/CMAKE_POLICY_VERSION_MINIMUM.html
# _LARGEFILE64_SOURCE fixes Abseil's missing off64_t declaration on Alpine/musl
RUN cmake \
        -S /tmp/nginx-otel \
        -B /tmp/nginx-otel/build \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_CXX_FLAGS=-D_LARGEFILE64_SOURCE \
        -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
        -DNGX_OTEL_NGINX_BUILD_DIR="${PWD}/objs"

RUN MODULE_BUILD_JOBS="${BUILD_JOBS:-$(nproc)}" \
    && cmake --build /tmp/nginx-otel/build \
        --config Release \
        --parallel "${MODULE_BUILD_JOBS}" \
    && mkdir -p /usr/local/openresty/nginx/modules \
    && cp objs/*.so /usr/local/openresty/nginx/modules/ \
    && cp /tmp/nginx-otel/build/ngx_otel_module.so /usr/local/openresty/nginx/modules/ \
    && strip --strip-unneeded /usr/local/openresty/nginx/modules/*.so

FROM ${OPENRESTY_IMAGE} AS runtime

RUN apk add --no-cache c-ares libgcc libmaxminddb-libs libssl3 libstdc++ zlib

COPY --from=modules-builder /usr/local/openresty/nginx/modules/ /usr/local/openresty/nginx/modules/
