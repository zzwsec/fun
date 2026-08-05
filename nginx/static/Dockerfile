FROM alpine AS builder
ARG NGINX_VERSION
ARG ENTRYPOINT_PATH=/usr/src/docker-nginx/entrypoint
ARG DEFAULT_CONFIG_PATH=/usr/src/pkg-oss/alpine/alpine
ARG CONFIG_PATH=/usr/src/nginx-${NGINX_VERSION}

RUN apk add --update --no-cache \
    git wget curl build-base automake autoconf libtool pkgconfig linux-headers \
    pcre2-dev zlib-dev openssl-dev brotli-dev zstd-dev zstd-static curl-dev cargo clang clang-dev

WORKDIR /usr/src

RUN git clone --depth=1 https://github.com/nginx/docker-nginx.git \
    && git clone --depth=1 -b ${NGINX_VERSION}-1 https://github.com/nginx/pkg-oss.git \
    && curl -fsSL https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz | tar xzf - -C /usr/src \
    && git clone --depth=1 https://github.com/openresty/headers-more-nginx-module.git nginx-${NGINX_VERSION}/ngx_headers_more \
    && git clone --depth=1 https://github.com/tokers/zstd-nginx-module.git nginx-${NGINX_VERSION}/ngx_zstd \
    && git clone --depth=1 https://github.com/nginx/nginx-acme.git nginx-${NGINX_VERSION}/ngx_http_acme_module \
    && git clone --depth=1 --recurse-submodules --shallow-submodules -j$(nproc) https://github.com/google/ngx_brotli nginx-${NGINX_VERSION}/ngx_brotli

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
    --add-module=ngx_brotli \
    && make -j$(nproc) \
    && make install \
    && strip /usr/sbin/nginx

RUN mkdir -p /tmp/exec/usr/sbin /tmp/exec/docker-entrypoint.d \
    && mkdir -p /tmp/conf/etc/nginx/conf.d /tmp/conf/usr/share/nginx/html \
    && cp /usr/sbin/nginx /tmp/exec/usr/sbin/ \
    && cp ${ENTRYPOINT_PATH}/docker-entrypoint.sh /tmp/exec/ \
    && cp ${ENTRYPOINT_PATH}/10-listen-on-ipv6-by-default.sh /tmp/exec/docker-entrypoint.d/ \
    && cp ${ENTRYPOINT_PATH}/15-local-resolvers.envsh /tmp/exec/docker-entrypoint.d/ \
    && cp ${ENTRYPOINT_PATH}/20-envsubst-on-templates.sh /tmp/exec/docker-entrypoint.d/ \
    && cp ${ENTRYPOINT_PATH}/30-tune-worker-processes.sh /tmp/exec/docker-entrypoint.d/ \
    && cp ${DEFAULT_CONFIG_PATH}/nginx.conf /tmp/conf/etc/nginx/ \
    && cp ${DEFAULT_CONFIG_PATH}/nginx.default.conf /tmp/conf/etc/nginx/conf.d/default.conf \
    && cp ${CONFIG_PATH}/conf/fastcgi_params /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/fastcgi.conf /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/mime.types /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/scgi_params /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/conf/uwsgi_params /tmp/conf/etc/nginx/ \
    && cp ${CONFIG_PATH}/html/index.html /tmp/conf/usr/share/nginx/html/ \
    && cp ${CONFIG_PATH}/html/50x.html /tmp/conf/usr/share/nginx/html/

FROM scratch

COPY --from=alpine / /

ARG NGINX_VERSION

RUN set -x \
    && apk add --update --no-cache pcre2 zlib openssl brotli-libs ca-certificates tzdata libgcc zstd-libs gettext-envsubst \
    && mkdir -p /var/cache/nginx /docker-entrypoint.d /var/log/nginx /usr/share/nginx/html /etc/nginx/conf.d /usr/lib/nginx/modules \
    && addgroup -g 101 -S nginx \
    && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
    && ln -sf /usr/lib/nginx/modules /etc/nginx/modules \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

COPY --chmod=755 --from=builder /tmp/exec/ /
COPY --from=builder /tmp/conf/ /

EXPOSE 80 443
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]