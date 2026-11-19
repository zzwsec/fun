# syntax=docker/dockerfile:1

ARG ALPINE_IMAGE="alpine:3.24"

FROM ${ALPINE_IMAGE} AS builder

ARG NGINX_VERSION="1.31.9"

ENV NGINX_SOURCE_DIR="/usr/src/nginx-${NGINX_VERSION}" \
    DOCKER_NGINX_ENTRYPOINT_DIR="/usr/src/docker-nginx/entrypoint" \
    PKG_OSS_ALPINE_CONFIG_DIR="/usr/src/pkg-oss/alpine/alpine" \
    NGX_ACME_STATE_PREFIX="/var/cache/nginx" \
    RUSTFLAGS="-Ctarget-feature=-crt-static"

RUN apk add --no-cache \
    build-base cargo clang clang-dev cmake curl gd-dev git libmaxminddb-dev libmaxminddb-static \
    libxml2-dev libxslt-dev linux-headers openssl-dev pcre2-dev pcre2-static pkgconfig \
    zlib-dev zlib-static zstd-dev zstd-static

WORKDIR /usr/src
RUN curl -fsSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" | tar xz --no-same-owner -C /usr/src \
    && git clone --depth=1 https://github.com/nginx/docker-nginx.git \
    && git clone --depth=1 https://github.com/openresty/headers-more-nginx-module.git "${NGINX_SOURCE_DIR}/ngx_headers_more" \
    && git clone --depth=1 https://github.com/tokers/zstd-nginx-module.git "${NGINX_SOURCE_DIR}/ngx_zstd" \
    && git clone --depth=1 https://github.com/nginx/nginx-acme.git "${NGINX_SOURCE_DIR}/ngx_http_acme_module" \
    && git clone --depth=1 https://github.com/nginx/njs.git "${NGINX_SOURCE_DIR}/njs" \
    && git clone --depth=1 --recurse-submodules --shallow-submodules --jobs="$(nproc)" https://github.com/google/ngx_brotli.git "${NGINX_SOURCE_DIR}/ngx_brotli" \
    && git clone --depth=1 --branch "${NGINX_VERSION}-1" https://github.com/nginx/pkg-oss.git

WORKDIR ${NGINX_SOURCE_DIR}/ngx_brotli/deps/brotli
RUN curl -fsSL https://salsa.debian.org/nginx-team/libnginx-mod-http-brotli/raw/debian/latest/debian/patches/0001-Fix-Vary-header.patch | \
        patch -d "${NGINX_SOURCE_DIR}/ngx_brotli" -p1 \
    && cmake -S . -B out \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_SHARED_LIBS=OFF \
        -DBROTLI_DISABLE_TESTS=ON \
        -DCMAKE_C_FLAGS="-Ofast -flto -funroll-loops -ffunction-sections -fdata-sections" \
    && cmake --build out \
        --config Release \
        --target brotlienc \
        --parallel "$(nproc)"

WORKDIR ${NGINX_SOURCE_DIR}
# Dynamic modules resolve PCRE2 and zlib symbols from the nginx executable.
RUN CORE_LIBS='-l:libmaxminddb.a -Wl,--whole-archive -l:libpcre2-8.a -l:libz.a -Wl,--no-whole-archive' ./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --modules-path=/usr/lib/nginx/modules \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/run/nginx.pid \
    --lock-path=/run/nginx.lock \
    --http-client-body-temp-path=/var/cache/nginx/client_temp \
    --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
    --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
    --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
    --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
    --user=nginx \
    --group=nginx \
    --with-compat --with-control-api --with-file-aio --with-threads \
    --without-select_module --without-poll_module \
    --with-http_addition_module --with-http_auth_request_module --with-http_geoip_module \
    --with-http_gunzip_module --with-http_gzip_static_module --with-http_json_module \
    --with-http_image_filter_module=dynamic \
    --with-http_random_index_module --with-http_realip_module \
    --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module \
    --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module \
    --with-stream --with-stream_geoip_module --with-stream_realip_module \
    --with-stream_ssl_module --with-stream_ssl_preread_module \
    --with-cc-opt='-Os -fstack-clash-protection -Wformat -Werror=format-security -fno-plt' \
    --with-ld-opt='-static-libgcc \
        -Wl,--exclude-libs,libgcc.a:libgcc_eh.a:libmaxminddb.a \
        -Wl,--as-needed,-O1,--sort-common,-z,pack-relative-relocs' \
    --add-module=ngx_headers_more \
    --add-module=ngx_zstd \
    --add-module=ngx_http_acme_module \
    --add-module=ngx_brotli \
    --add-dynamic-module=njs/nginx \
    && make -j"$(nproc)" \
    && strip objs/nginx objs/*.so

RUN mkdir -p /etc/nginx/conf.d /usr/share/nginx /usr/sbin /usr/lib/nginx/modules /docker-entrypoint.d \
    && cp -a "${NGINX_SOURCE_DIR}/objs/nginx" /usr/sbin/nginx \
    && cp -a "${NGINX_SOURCE_DIR}"/objs/*.so /usr/lib/nginx/modules/ \
    && cp -a "${NGINX_SOURCE_DIR}/html" /usr/share/nginx/ \
    && cp -a "${NGINX_SOURCE_DIR}/conf/." /etc/nginx/ \
    && cp "${PKG_OSS_ALPINE_CONFIG_DIR}/nginx.conf" /etc/nginx/nginx.conf \
    && cp "${PKG_OSS_ALPINE_CONFIG_DIR}/nginx.default.conf" /etc/nginx/conf.d/default.conf \
    && rm -f /etc/nginx/koi-* /etc/nginx/win-utf \
    && cp -a "${DOCKER_NGINX_ENTRYPOINT_DIR}/." /docker-entrypoint.d/ \
    && mv /docker-entrypoint.d/docker-entrypoint.sh /docker-entrypoint.sh

FROM ${ALPINE_IMAGE} AS static

COPY --from=builder /etc/nginx/ /etc/nginx/
COPY --from=builder /usr/share/nginx/ /usr/share/nginx/
COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /docker-entrypoint.sh /docker-entrypoint.sh
COPY --from=builder /docker-entrypoint.d/ /docker-entrypoint.d/

RUN set -x \
    && apk add --no-cache gettext-envsubst libssl3 tzdata \
    && mkdir -p /usr/lib/nginx/modules /var/cache/nginx /var/log/nginx \
    && addgroup -g 101 -S nginx \
    && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
    && ln -sf /usr/lib/nginx/modules /etc/nginx/modules \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log \
    && nginx -t

EXPOSE 80/tcp 443/tcp 443/udp
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]

FROM static AS dynamic

RUN apk add --no-cache libgd libxml2
COPY --from=builder /usr/lib/nginx/modules/*.so /usr/lib/nginx/modules/
