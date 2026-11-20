# syntax=docker/dockerfile:1

ARG ALPINE_VERSION=3.24
ARG PHP_INI_DIR=/usr/local/etc/php

FROM alpine:${ALPINE_VERSION} AS builder

ARG PHP_INI_DIR
ARG PHP_VERSION=8.5.13
ARG PHPREDIS_VERSION=6.3.0
ARG PHP_URL="https://www.php.net/distributions/php-${PHP_VERSION}.tar.xz"

# Install build toolchain and extension development headers, then fetch PHP source.
# Remove musl's iconv.h so PHP links against GNU libiconv (full encoding/translit support).
WORKDIR /usr/src/php
RUN set -eux -o pipefail \
    && apk add --no-cache \
        autoconf \
        argon2-dev \
        bzip2-dev \
        ca-certificates \
        coreutils \
        curl \
        curl-dev \
        file \
        freetype-dev \
        g++ \
        gcc \
        gettext-dev \
        gmp-dev \
        gnu-libiconv-dev \
        icu-dev \
        libc-dev \
        libjpeg-turbo-dev \
        libpng-dev \
        libsodium-dev \
        libwebp-dev \
        libxml2-dev \
        libxslt-dev \
        libzip-dev \
        linux-headers \
        make \
        oniguruma-dev \
        openldap-dev \
        openssl-dev \
        pkgconf \
        re2c \
        readline-dev \
        sqlite-dev \
        tar \
        unixodbc-dev \
        xz \
        zlib-dev \
    && curl -fsSL "$PHP_URL" | tar -xJf - --strip-components=1 \
    && rm -f /usr/include/iconv.h

# Compile PHP with hardening flags scoped only to the main PHP build.
# Explicit --prefix=/usr/local prevents "NONE/" paths in php-fpm.conf.default.
RUN set -eux -o pipefail \
    && export \
        CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64" \
        CPPFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64" \
        LDFLAGS="-Wl,-O1 -Wl,-z,relro,-z,now -pie" \
        PHP_UNAME="Linux - Docker" \
    && ./configure \
        --build="$(gcc -dumpmachine)" \
        --prefix=/usr/local \
        --sysconfdir="${PHP_INI_DIR%/php}" \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        --enable-option-checking=fatal \
        --disable-cgi \
        --disable-phpdbg \
        --enable-bcmath \
        --enable-calendar \
        --enable-exif \
        --enable-fpm \
        --enable-gd \
        --enable-intl \
        --enable-mbstring \
        --enable-mysqlnd \
        --enable-pcntl \
        --enable-shmop \
        --enable-soap \
        --with-bz2 \
        --with-curl \
        --with-fpm-group=www-data \
        --with-fpm-user=www-data \
        --with-freetype \
        --with-gettext \
        --with-gmp \
        --with-iconv=/usr \
        --with-jpeg \
        --with-ldap \
        --with-mysqli \
        --with-openssl \
        --with-password-argon2 \
        --with-pdo-mysql \
        --with-pdo-sqlite \
        --with-pic \
        --with-readline \
        --with-sodium \
        --with-sqlite3 \
        --with-unixODBC \
        --with-webp \
        --with-xsl \
        --with-zip \
        --with-zlib \
    && make -j"$(nproc)" \
    && find . -type f -name '*.a' -delete \
    && make install


# Compile phpredis as a shared module using PIC/RELRO flags
WORKDIR /usr/src/phpredis
RUN set -eux -o pipefail \
    && curl -fsSL "https://codeload.github.com/phpredis/phpredis/tar.gz/refs/tags/${PHPREDIS_VERSION}" \
        | tar -xzf - --strip-components=1 \
    && export \
        CFLAGS="-fstack-protector-strong -fpic -O2" \
        CPPFLAGS="-fstack-protector-strong -fpic -O2" \
        LDFLAGS="-Wl,-O1 -Wl,-z,relro,-z,now" \
    && phpize \
    && ./configure --with-php-config=/usr/local/bin/php-config \
    && make -j"$(nproc)" \
    && make install

# Prune build-only files from /usr/local, strip ELF binaries/extensions, and install production php.ini.
RUN set -eux -o pipefail \
    && rm -rf \
        /usr/local/bin/php-config \
        /usr/local/bin/phpize \
        /usr/local/include/php \
        /usr/local/lib/php/build \
        /usr/local/php/man \
    && find /usr/local -type f \( -perm /0111 -o -name '*.so' \) -exec sh -c ' \
        for f; do \
            if file "$f" | grep -q "ELF"; then \
                strip --strip-unneeded "$f"; \
            fi; \
        done \
    ' sh {} + \
    && mkdir -p "${PHP_INI_DIR}/conf.d" \
    && cp /usr/src/php/php.ini-production "${PHP_INI_DIR}/php.ini"

WORKDIR /usr/local/etc

# Generate FPM entrypoint, FPM pool configs, and modular PHP INI files.
RUN <<'EOF'
set -eux

# Inline FPM-specific entrypoint (avoids fetching unpinned scripts from GitHub master).
cat <<'INNER_EOF' > /usr/local/bin/docker-php-entrypoint
#!/bin/sh
set -e

# Prepend php-fpm if the first argument is a flag (e.g. -t or -F).
if [ "${1#-}" != "$1" ]; then
    set -- php-fpm "$@"
fi

exec "$@"
INNER_EOF
chmod +x /usr/local/bin/docker-php-entrypoint

mv php-fpm.conf.default php-fpm.conf
rm -f php-fpm.d/*.conf php-fpm.d/www.conf.default

cat <<'INNER_EOF' > php-fpm.d/00-global.conf
[global]
error_log = /proc/self/fd/2

; Raise log line limit to prevent truncating long stack traces.
log_limit = 8192

; Run PHP-FPM in the foreground as PID 1.
daemonize = no
INNER_EOF

cat <<'INNER_EOF' > php-fpm.d/10-www.conf
[www]
; Container runs directly as non-root www-data, so user/group directives are omitted.

; Listen on all container interfaces; host exposure is controlled by Docker/orchestrator.
listen = 9000
listen.backlog = 1024

pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 5

; Recycle workers periodically to mitigate memory growth in extensions or libraries.
pm.max_requests = 500

; Hard FPM-level timeout for stuck requests (complements PHP max_execution_time).
request_terminate_timeout = 60s

; Log PHP backtraces for requests slower than 5 seconds.
request_slowlog_timeout = 5s
request_slowlog_trace_depth = 20
slowlog = /proc/self/fd/2

; PHP-FPM closes stdout on startup; route access logs to stderr instead.
access.log = /proc/self/fd/2

; Preserve container environment variables for PHP workers.
clear_env = no

; Forward worker stdout/stderr to the main error log without extra prefixing.
catch_workers_output = yes
decorate_workers_output = no

; Restrict FPM execution strictly to .php scripts.
security.limit_extensions = .php
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/00-fpm.ini"
; Avoid duplicate error logs when catch_workers_output is enabled.
fastcgi.logging = Off
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/10-opcache.ini"
; OPcache is built into PHP core as of PHP 8.5 (no zend_extension line needed).
[opcache]
opcache.enable = 1
opcache.enable_cli = 0

; Shared opcode memory across all PHP-FPM workers.
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000

; Disable filesystem timestamp checks for immutable container deployments.
opcache.validate_timestamps = 0

; Retain PHPDoc comments required by frameworks, annotations, and reflection.
opcache.save_comments = 1

; Explicitly disable JIT for typical I/O-bound web workloads (cross-version safe).
opcache.jit = disable
opcache.jit_buffer_size = 0
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/15-performance.ini"
memory_limit = 128M

; Extend realpath cache TTL for immutable container filesystems (default size is already 4096K).
realpath_cache_ttl = 600
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/20-security.ini"
cgi.fix_pathinfo = 0
expose_php = Off
display_errors = Off
display_startup_errors = Off
allow_url_fopen = Off

; Session hardening
session.use_strict_mode = 1
session.use_only_cookies = 1
session.cookie_httponly = 1
session.cookie_samesite = Lax
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/30-timezone.ini"
date.timezone = ${TZ}
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/40-redis.ini"
extension = redis.so
INNER_EOF
EOF

FROM alpine:${ALPINE_VERSION}

ARG PHP_INI_DIR

ENV TZ=Asia/Shanghai \
    PHP_INI_DIR=${PHP_INI_DIR}

WORKDIR /var/www/html

COPY --from=builder /usr/local/ /usr/local/

RUN set -eux -o pipefail \
    && adduser -u 82 -D -S -G www-data www-data \
    && chown www-data:www-data /var/www/html \
    && run_deps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
            | tr ',' '\n' \
            | sort -u \
            | awk 'NF && system("[ -e /usr/local/lib/" $1 " ]") != 0 { print "so:" $1 }' \
    )" \
    && apk add --no-cache $run_deps ca-certificates tzdata

USER www-data

# Validate FPM configuration under the actual runtime user.
RUN php-fpm -t

EXPOSE 9000
STOPSIGNAL SIGQUIT
ENTRYPOINT ["docker-php-entrypoint"]
CMD ["php-fpm"]