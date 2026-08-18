# syntax=docker/dockerfile:1

FROM alpine:3.24

ARG NGINX_VERSION="1.31.3"

RUN set -eux \
    && NGINX_SOURCE_DIR="/usr/src/nginx-${NGINX_VERSION}" \
    && DOCKER_NGINX_ENTRYPOINT_DIR="/usr/src/docker-nginx/entrypoint" \
    && PKG_OSS_ALPINE_CONFIG_DIR="/usr/src/pkg-oss/alpine/alpine" \
    && apk add --no-cache \
        ca-certificates \
        gettext-envsubst \
        tzdata \
    && apk add --no-cache --virtual .build-deps \
        build-base \
        curl \
        git \
        linux-headers \
        openssl-dev \
        pax-utils \
        pcre2-dev \
        zlib-dev \
    && mkdir -p /usr/src \
    && curl -fsSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" | tar xzf - -C /usr/src \
    && git clone --depth=1 https://github.com/nginx/docker-nginx.git /usr/src/docker-nginx \
    && git clone --depth=1 https://github.com/nginx/pkg-oss.git /usr/src/pkg-oss \
    && cd "${NGINX_SOURCE_DIR}" \
    && ./configure \
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
        --with-debug \
        --with-compat \
        --with-file-aio \
        --with-threads \
        --without-select_module \
        --without-poll_module \
        --with-http_addition_module \
        --with-http_auth_request_module \
        --with-http_gunzip_module \
        --with-http_gzip_static_module \
        --with-http_random_index_module \
        --with-http_realip_module \
        --with-http_secure_link_module \
        --with-http_slice_module \
        --with-http_ssl_module \
        --with-http_stub_status_module \
        --with-http_sub_module \
        --with-http_v2_module \
        --with-http_v3_module \
        --with-stream \
        --with-stream_realip_module \
        --with-stream_ssl_module \
        --with-stream_ssl_preread_module \
        --with-cc-opt="-Os -g -fstack-clash-protection -Wformat -Werror=format-security -fno-plt" \
        --with-ld-opt="-Wl,--as-needed,-O1,--sort-common -Wl,-z,pack-relative-relocs" \
    && make -j"$(nproc)" \
    && make install \
    && mkdir -p /etc/nginx/conf.d /usr/lib/nginx/modules /usr/share/nginx/html /var/cache/nginx /var/log/nginx \
    && cp "${PKG_OSS_ALPINE_CONFIG_DIR}/nginx.conf" /etc/nginx/nginx.conf \
    && cp "${PKG_OSS_ALPINE_CONFIG_DIR}/nginx.default.conf" /etc/nginx/conf.d/default.conf \
    && sed -i 's#notice#debug#' /etc/nginx/nginx.conf \
    && cp -a "${NGINX_SOURCE_DIR}/html/." /usr/share/nginx/html/ \
    && rm -rf /etc/nginx/html /etc/nginx/*.default /etc/nginx/koi-* /etc/nginx/win-utf \
    && mv "${DOCKER_NGINX_ENTRYPOINT_DIR}/docker-entrypoint.sh" /docker-entrypoint.sh \
    && mv "${DOCKER_NGINX_ENTRYPOINT_DIR}" /docker-entrypoint.d \
    && addgroup -g 101 -S nginx \
    && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
    && ln -sf /usr/lib/nginx/modules /etc/nginx/modules \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' /usr/sbin/nginx \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --no-cache --virtual .nginx-rundeps ${runDeps} \
    && apk del --no-network .build-deps \
    && rm -rf /usr/src \
    && nginx -V \
    && nginx -t

EXPOSE 80/tcp 443/tcp 443/udp
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]