# syntax=docker/dockerfile:1

ARG ALPINE_IMAGE="alpine:3.24"

FROM ${ALPINE_IMAGE} AS builder

ARG CORE_RULESET_VERSION="4.30.0"
ARG MODSECURITY_NGINX_VERSION="1.0.4"
ARG MODSECURITY_VERSION="3.0.17"
ARG NGINX_VERSION="1.31.9"
ARG WORDPRESS_RULE_EXCLUSIONS_VERSION="1.2.0"

ENV CORE_RULESET_SOURCE_DIR="/usr/src/coreruleset" \
    DOCKER_NGINX_ENTRYPOINT_DIR="/usr/src/docker-nginx/entrypoint" \
    MODSECURITY_SOURCE_DIR="/usr/src/ModSecurity" \
    NGINX_SOURCE_DIR="/usr/src/nginx-${NGINX_VERSION}" \
    PKG_OSS_ALPINE_CONFIG_DIR="/usr/src/pkg-oss/alpine/alpine" \
    WORDPRESS_RULE_EXCLUSIONS_SOURCE_DIR="/usr/src/wordpress-rule-exclusions-plugin" \
    NGX_ACME_STATE_PREFIX="/var/cache/nginx" \
    RUSTFLAGS="-Ctarget-feature=-crt-static"

RUN apk add --no-cache \
    autoconf automake build-base cargo clang clang-dev cmake curl curl-dev git \
    libmaxminddb-dev libmaxminddb-static libtool libxml2-dev linux-headers \
    openssl-dev pcre2-dev pcre2-static pkgconfig yajl-dev zlib-dev zlib-static zstd-dev zstd-static

WORKDIR /usr/src
RUN curl -fsSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" | tar xz --no-same-owner -C /usr/src \
    && git clone --depth=1 --branch "v${CORE_RULESET_VERSION}" https://github.com/coreruleset/coreruleset.git \
    && git clone --depth=1 --branch "v${MODSECURITY_VERSION}" --recurse-submodules --shallow-submodules https://github.com/owasp-modsecurity/ModSecurity.git \
    && git clone --depth=1 --branch "v${MODSECURITY_NGINX_VERSION}" https://github.com/owasp-modsecurity/ModSecurity-nginx.git "${NGINX_SOURCE_DIR}/ngx_modsecurity" \
    && git clone --depth=1 --branch "v${WORDPRESS_RULE_EXCLUSIONS_VERSION}" https://github.com/coreruleset/wordpress-rule-exclusions-plugin.git \
    && git clone --depth=1 https://github.com/nginx/docker-nginx.git \
    && git clone --depth=1 --branch "${NGINX_VERSION}-1" https://github.com/nginx/pkg-oss.git \
    && git clone --depth=1 https://github.com/openresty/headers-more-nginx-module.git "${NGINX_SOURCE_DIR}/ngx_headers_more" \
    && git clone --depth=1 https://github.com/tokers/zstd-nginx-module.git "${NGINX_SOURCE_DIR}/ngx_zstd" \
    && git clone --depth=1 https://github.com/nginx/nginx-acme.git "${NGINX_SOURCE_DIR}/ngx_http_acme_module" \
    && git clone --depth=1 --recurse-submodules --shallow-submodules --jobs="$(nproc)" https://github.com/google/ngx_brotli.git "${NGINX_SOURCE_DIR}/ngx_brotli"

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

WORKDIR ${MODSECURITY_SOURCE_DIR}
RUN ./build.sh \
    && ./configure --disable-shared --enable-static \
    && make -j"$(nproc)" \
    && make install

WORKDIR /etc/nginx/modsec
RUN mkdir -p plugins rules \
    && cp "${MODSECURITY_SOURCE_DIR}/unicode.mapping" ./ \
    && cp "${MODSECURITY_SOURCE_DIR}/modsecurity.conf-recommended" modsecurity.conf \
    && cp "${WORDPRESS_RULE_EXCLUSIONS_SOURCE_DIR}/plugins/wordpress-rule-exclusions-before.conf" \
        "${WORDPRESS_RULE_EXCLUSIONS_SOURCE_DIR}/plugins/wordpress-rule-exclusions-config.conf" plugins/ \
    && cp -a "${CORE_RULESET_SOURCE_DIR}/rules/." rules/ \
    && cp "${CORE_RULESET_SOURCE_DIR}/crs-setup.conf.example" crs-setup.conf \
    && touch plugins/placeholder-after.conf \
    && sed -i \
        -e 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' \
        -e 's/SecPcreMatchLimit [0-9]\+/SecPcreMatchLimit 20000/' \
        -e 's/SecPcreMatchLimitRecursion [0-9]\+/SecPcreMatchLimitRecursion 20000/' \
        -e 's/^SecRequestBodyLimit\s\+[0-9]\+/SecRequestBodyLimit 52428800/' \
        -e 's/^SecRequestBodyNoFilesLimit\s\+[0-9]\+/SecRequestBodyNoFilesLimit 524288/' \
        -e 's/^SecAuditEngine RelevantOnly/SecAuditEngine Off/' \
        modsecurity.conf \
    && printf '%s\n' \
        'SecAction "id:900110,phase:1,pass,t:none,nolog,setvar:tx.inbound_anomaly_score_threshold=30,setvar:tx.outbound_anomaly_score_threshold=16"' \
        > plugins/anomaly-threshold-config.conf \
    && printf '%s\n' \
        'Include /etc/nginx/modsec/crs-setup.conf' \
        'Include /etc/nginx/modsec/plugins/*-config.conf' \
        'Include /etc/nginx/modsec/plugins/*-before.conf' \
        'Include /etc/nginx/modsec/rules/*.conf' \
        'Include /etc/nginx/modsec/plugins/*-after.conf' \
        >> modsecurity.conf

WORKDIR ${NGINX_SOURCE_DIR}
RUN sed -i \
        's|-lmodsecurity|/usr/local/modsecurity/lib/libmodsecurity.a -lcurl -lxml2 -l:libpcre2-8.a -l:libmaxminddb.a -lyajl -lstdc++ -lm -lrt -pthread|g' \
        ngx_modsecurity/config \
    && sed -i \
        -e 's|ngx_zstd_opt_L="-l:libzstd.a"|ngx_zstd_opt_L="-Wl,--whole-archive -l:libzstd.a -Wl,--no-whole-archive"|' \
        -e 's|NGX_LD_OPT="$ngx_zstd_opt_L $NGX_LD_OPT"|:|' \
        ngx_zstd/filter/config ngx_zstd/static/config \
    && sed -i \
        's|ngx_module_libs=$NGX_LD_OPT|ngx_module_libs="$ngx_zstd_opt_L"|' \
        ngx_zstd/filter/config \
    && sed -i \
        '/ngx_module_srcs=$HTTP_ZSTD_SRCS/a ngx_module_libs=' \
        ngx_zstd/static/config \
    && NGX_IGNORE_RPATH=YES ./configure \
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
    --with-http_random_index_module --with-http_realip_module \
    --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module \
    --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module \
    --with-stream --with-stream_geoip_module --with-stream_realip_module \
    --with-stream_ssl_module --with-stream_ssl_preread_module \
    --with-cc-opt='-Os -fstack-clash-protection -Wformat -Werror=format-security -fno-plt -g' \
    --with-ld-opt="-shared-libgcc \
        -Wl,--as-needed,-O1,--sort-common,-z,pack-relative-relocs \
        -Wl,--exclude-libs,libbrotlicommon.a:libbrotlienc.a:libmaxminddb.a:libpcre2-8.a:libz.a:libzstd.a \
        -l:libmaxminddb.a -l:libpcre2-8.a -l:libz.a" \
    --add-module=ngx_headers_more \
    --add-module=ngx_zstd \
    --add-module=ngx_http_acme_module \
    --add-module=ngx_modsecurity \
    --add-module=ngx_brotli \
    && make -j"$(nproc)" \
    && strip objs/nginx

RUN mkdir -p /etc/nginx/conf.d /usr/share/nginx /usr/sbin /docker-entrypoint.d \
    && cp -a "${NGINX_SOURCE_DIR}/objs/nginx" /usr/sbin/nginx \
    && cp -a "${NGINX_SOURCE_DIR}/html" /usr/share/nginx/ \
    && cp -a "${NGINX_SOURCE_DIR}/conf/." /etc/nginx/ \
    && cp "${PKG_OSS_ALPINE_CONFIG_DIR}/nginx.conf" /etc/nginx/nginx.conf \
    && cp "${PKG_OSS_ALPINE_CONFIG_DIR}/nginx.default.conf" /etc/nginx/conf.d/default.conf \
    && rm -f /etc/nginx/koi-* /etc/nginx/win-utf \
    && cp -a "${DOCKER_NGINX_ENTRYPOINT_DIR}/." /docker-entrypoint.d/ \
    && mv /docker-entrypoint.d/docker-entrypoint.sh /docker-entrypoint.sh

FROM ${ALPINE_IMAGE}

COPY --from=builder /etc/nginx/ /etc/nginx/
COPY --from=builder /usr/share/nginx/ /usr/share/nginx/
COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /docker-entrypoint.sh /docker-entrypoint.sh
COPY --from=builder /docker-entrypoint.d/ /docker-entrypoint.d/

RUN set -x \
    && apk add --no-cache ca-certificates gettext-envsubst libcurl libssl3 libstdc++ libxml2 tzdata yajl \
    && mkdir -p /usr/lib/nginx/modules /var/cache/nginx /var/log/nginx \
    && addgroup -g 101 -S nginx \
    && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
    && ln -sf /usr/lib/nginx/modules /etc/nginx/modules \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80/tcp 443/tcp 443/udp
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
