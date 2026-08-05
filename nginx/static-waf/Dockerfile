FROM alpine AS builder
ARG NGINX_VERSION
ARG MODSEC_VERSION
ARG ENTRYPOINT_PATH=/usr/src/docker-nginx/entrypoint
ARG RULESET_PATH=/usr/src/coreruleset
ARG DEFAULT_CONFIG_PATH=/usr/src/pkg-oss/alpine/alpine
ARG CONFIG_PATH=/usr/src/nginx-${NGINX_VERSION}

RUN apk add --update --no-cache \
    git wget curl build-base automake autoconf libtool pkgconfig linux-headers \
    pcre2-dev zlib-dev openssl-dev brotli-dev zstd-dev zstd-static \
    libxml2-dev curl-dev yajl-dev lmdb-dev cargo clang clang-dev

WORKDIR /usr/src

RUN git clone --depth 1 https://github.com/coreruleset/coreruleset.git \
    && git clone --depth=1 -b v${MODSEC_VERSION} --recurse-submodules --shallow-submodules https://github.com/owasp-modsecurity/ModSecurity \
    && git clone --depth=1 https://github.com/nginx/docker-nginx.git \
    && git clone --depth=1 -b ${NGINX_VERSION}-1 https://github.com/nginx/pkg-oss.git \
    && curl -fsSL https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz | tar xzf - -C /usr/src \
    && git clone --depth=1 https://github.com/openresty/headers-more-nginx-module.git nginx-${NGINX_VERSION}/ngx_headers_more \
    && git clone --depth=1 https://github.com/tokers/zstd-nginx-module.git nginx-${NGINX_VERSION}/ngx_zstd \
    && git clone --depth=1 https://github.com/nginx/nginx-acme.git nginx-${NGINX_VERSION}/ngx_http_acme_module \
    && git clone --depth=1 https://github.com/owasp-modsecurity/ModSecurity-nginx.git nginx-${NGINX_VERSION}/ngx_modsecurity \
    && git clone --depth=1 --recurse-submodules --shallow-submodules -j$(nproc) https://github.com/google/ngx_brotli nginx-${NGINX_VERSION}/ngx_brotli

WORKDIR /usr/src/ModSecurity
RUN ./build.sh \
    && ./configure CXXFLAGS="-include cstdint" \
    && make -j$(nproc) \
    && make install \
    && mkdir -p /tmp/plugins \
    && wget -P /tmp/plugins https://fastly.jsdelivr.net/gh/coreruleset/wordpress-rule-exclusions-plugin@master/plugins/wordpress-rule-exclusions-before.conf \
    && wget -P /tmp/plugins https://fastly.jsdelivr.net/gh/coreruleset/wordpress-rule-exclusions-plugin@master/plugins/wordpress-rule-exclusions-config.conf \
    && wget -P /tmp/plugins https://fastly.jsdelivr.net/gh/owasp-modsecurity/ModSecurity@v3/unicode.mapping \
    && wget -O /tmp/plugins/modsecurity.conf https://fastly.jsdelivr.net/gh/owasp-modsecurity/ModSecurity@v3/modsecurity.conf-recommended \
    && sed -i 's#setvar:tx.inbound_anomaly_score_threshold=5#setvar:tx.inbound_anomaly_score_threshold=30#' /usr/src/coreruleset/crs-setup.conf.example \
    && sed -i 's#setvar:tx.outbound_anomaly_score_threshold=4#setvar:tx.outbound_anomaly_score_threshold=16#' /usr/src/coreruleset/crs-setup.conf.example \
    && sed -i \
        -e 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' \
        -e 's/SecPcreMatchLimit [0-9]\+/SecPcreMatchLimit 20000/' \
        -e 's/SecPcreMatchLimitRecursion [0-9]\+/SecPcreMatchLimitRecursion 20000/' \
        -e 's/^SecRequestBodyLimit\s\+[0-9]\+/SecRequestBodyLimit 52428800/' \
        -e 's/^SecRequestBodyNoFilesLimit\s\+[0-9]\+/SecRequestBodyNoFilesLimit 524288/' \
        -e 's/^SecAuditEngine RelevantOnly/SecAuditEngine Off/' \
        /tmp/plugins/modsecurity.conf \
    && echo 'Include /etc/nginx/modsec/crs-setup.conf' >> /tmp/plugins/modsecurity.conf \
    && echo 'Include /etc/nginx/modsec/plugins/*-config.conf' >> /tmp/plugins/modsecurity.conf \
    && echo 'Include /etc/nginx/modsec/plugins/*-before.conf' >> /tmp/plugins/modsecurity.conf \
    && echo 'Include /etc/nginx/modsec/rules/*.conf' >> /tmp/plugins/modsecurity.conf \
    && echo 'Include /etc/nginx/modsec/plugins/*-after.conf' >> /tmp/plugins/modsecurity.conf


WORKDIR /usr/src/nginx-${NGINX_VERSION}
RUN ./configure \
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
    --with-compat --with-file-aio --with-threads \
    --with-http_addition_module --with-http_auth_request_module --with-http_dav_module \
    --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module \
    --with-http_mp4_module --with-http_random_index_module --with-http_realip_module \
    --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module \
    --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module \
    --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module \
    --with-cc-opt='-Os -fstack-clash-protection -Wformat -Werror=format-security -fno-plt -g' \
    --with-ld-opt='-Wl,--as-needed,-O1,--sort-common -Wl,-z,pack-relative-relocs' \
    --add-module=ngx_headers_more \
    --add-module=ngx_zstd \
    --add-module=ngx_http_acme_module \
    --add-module=ngx_modsecurity \
    --add-module=ngx_brotli \
    && make -j$(nproc) \
    && make install \
    && strip /usr/sbin/nginx \
    && strip /usr/local/modsecurity/lib/libmodsecurity.so.*

RUN mkdir -p /tmp/exec/usr/sbin /tmp/exec/docker-entrypoint.d \
    && mkdir -p /tmp/conf/usr/lib /tmp/conf/etc/nginx/modsec/plugins /tmp/conf/etc/nginx/modsec/rules /tmp/conf/etc/nginx/conf.d /tmp/conf/usr/share/nginx/html \
    && cp /usr/sbin/nginx /tmp/exec/usr/sbin/ \
    && cp ${ENTRYPOINT_PATH}/docker-entrypoint.sh /tmp/exec/ \
    && cp ${ENTRYPOINT_PATH}/10-listen-on-ipv6-by-default.sh /tmp/exec/docker-entrypoint.d/ \
    && cp ${ENTRYPOINT_PATH}/15-local-resolvers.envsh /tmp/exec/docker-entrypoint.d/ \
    && cp ${ENTRYPOINT_PATH}/20-envsubst-on-templates.sh /tmp/exec/docker-entrypoint.d/ \
    && cp ${ENTRYPOINT_PATH}/30-tune-worker-processes.sh /tmp/exec/docker-entrypoint.d/ \
    && cp /usr/local/modsecurity/lib/libmodsecurity.so.${MODSEC_VERSION} /tmp/conf/usr/lib/ \
    && cp /tmp/plugins/wordpress-rule-exclusions-before.conf /tmp/conf/etc/nginx/modsec/plugins/ \
    && cp /tmp/plugins/wordpress-rule-exclusions-config.conf /tmp/conf/etc/nginx/modsec/plugins/ \
    && touch /tmp/conf/etc/nginx/modsec/plugins/placeholder-after.conf \
    && cp /tmp/plugins/unicode.mapping /tmp/conf/etc/nginx/modsec/ \
    && cp /tmp/plugins/modsecurity.conf /tmp/conf/etc/nginx/modsec/modsecurity.conf \
    && cp ${DEFAULT_CONFIG_PATH}/nginx.conf /tmp/conf/etc/nginx/ \
    && cp ${DEFAULT_CONFIG_PATH}/nginx.default.conf /tmp/conf/etc/nginx/conf.d/default.conf \
    && cp -r ${RULESET_PATH}/rules/* /tmp/conf/etc/nginx/modsec/rules/ \
    && cp ${RULESET_PATH}/crs-setup.conf.example /tmp/conf/etc/nginx/modsec/crs-setup.conf \
    && cp ${CONFIG_PATH}/conf/fastcgi_params /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/fastcgi.conf /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/mime.types /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/scgi_params /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/uwsgi_params /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/html/index.html /tmp/conf/usr/share/nginx/html/ \
    && cp ${CONFIG_PATH}/html/50x.html /tmp/conf/usr/share/nginx/html/

FROM scratch

COPY --from=alpine:latest / /

ARG MODSEC_VERSION
ARG NGINX_VERSION

RUN set -x \
    && apk add --update --no-cache pcre2 zlib openssl brotli-libs libxml2 yajl lmdb libcurl libstdc++ ca-certificates tzdata libgcc zstd-libs gettext-envsubst \
    && mkdir -p /var/cache/nginx /docker-entrypoint.d /var/log/nginx /usr/share/nginx/html /etc/nginx/conf.d /etc/nginx/modsec/plugins /etc/nginx/modsec/rules /usr/lib/nginx/modules \
    && addgroup -g 101 -S nginx \
    && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
    && ln -sf /usr/lib/nginx/modules /etc/nginx/modules \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log \
    && ln -sf /usr/lib/libmodsecurity.so.${MODSEC_VERSION} /usr/lib/libmodsecurity.so.3 \
    && ln -sf /usr/lib/libmodsecurity.so.${MODSEC_VERSION} /usr/lib/libmodsecurity.so

COPY --chmod=755 --from=builder /tmp/exec/ /
COPY --from=builder /tmp/conf/ /

EXPOSE 80 443
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]