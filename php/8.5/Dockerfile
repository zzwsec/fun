# syntax=docker/dockerfile:1

FROM alpine:3.24 AS builder

ARG PHP_VERSION=8.5.9
ARG PHPREDIS_VERSION=6.3.0
ARG PHP_URL="https://www.php.net/distributions/php-${PHP_VERSION}.tar.xz"
ARG PHPIZE_DEPS="git ca-certificates curl openssl tar xz autoconf dpkg-dev dpkg file g++ gcc libc-dev make pkgconf re2c argon2-dev coreutils curl-dev gnu-libiconv-dev libsodium-dev libxml2-dev linux-headers oniguruma-dev openssl-dev patch patchutils readline-dev sqlite-dev bzip2-dev freetype-dev libjpeg-turbo-dev libpng-dev libwebp-dev gettext-dev gmp-dev openldap-dev unixodbc-dev libxslt-dev libzip-dev icu-dev"
ARG PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ARG PHP_CPPFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ARG PHP_LDFLAGS="-Wl,-O1 -pie"
ARG CFLAGS="$PHP_CFLAGS"
ARG CPPFLAGS="$PHP_CPPFLAGS"
ARG LDFLAGS="$PHP_LDFLAGS"

ARG PHP_INI_DIR=/usr/local/etc/php
ARG PHP_BUILD_PROVIDER='https://github.com/docker-library/php'
ARG PHP_UNAME='Linux - Docker'

RUN set -eux \
    && apk add --no-cache $PHPIZE_DEPS \
    && curl -fsSL "$PHP_URL" | tar Jxf - -C /usr/ \
    && rm -f /usr/include/iconv.h

WORKDIR /usr/php-${PHP_VERSION}

RUN set -eux \
    && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
    && ./configure \
        --build="$gnuArch" \
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
        --enable-option-checking=fatal \
        --enable-pcntl \
        --enable-shmop \
        --enable-soap \
        --sysconfdir="${PHP_INI_DIR%/php}" \
        --with-bz2 \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        --with-curl \
        --with-fpm-group=www-data \
        --with-fpm-user=www-data \
        --with-freetype \
        --with-gettext \
        --with-gmp \
        --with-iconv=/usr \
        --with-jpeg \
        --with-ldap \
        --with-mhash \
        --with-mysqli \
        --with-openssl \
        --with-password-argon2 \
        --with-pdo-mysql \
        --with-pdo-sqlite=/usr \
        --with-pear \
        --with-pic \
        --with-readline \
        --with-sodium=shared \
        --with-sqlite3=/usr \
        --with-unixODBC \
        --with-webp \
        --with-xsl \
        --with-zlib \
        --with-zip \
    && make -j$(nproc) \
    && make install

RUN set -eux \
    && git clone --depth 1 --branch "${PHPREDIS_VERSION}" https://github.com/phpredis/phpredis.git /usr/phpredis \
    && cd /usr/phpredis \
    && phpize \
    && ./configure --with-php-config=/usr/local/bin/php-config \
    && make -j$(nproc) \
    && make install

RUN set -eux \
    && find /usr/local/ -type f -perm '/0111' -exec sh -c ' \
        for f; do \
            if file "$f" | grep -q "ELF"; then \
                strip --strip-unneeded "$f"; \
            fi \
        done \
        ' sh {} + \
    && mkdir -p "${PHP_INI_DIR}/conf.d" \
    && cp /usr/php-${PHP_VERSION}/php.ini-production "${PHP_INI_DIR}/php.ini" \
    && curl -fsSL https://github.com/docker-library/php/archive/refs/heads/master.tar.gz | \
            tar zxf - -C /usr/local/bin/ php-master/docker-php-entrypoint php-master/docker-php-ext-configure php-master/docker-php-ext-enable php-master/docker-php-ext-install --strip-components=1

WORKDIR /usr/local/etc

RUN <<EOF
set -eux

sed 's|=NONE/|=|g' php-fpm.conf.default > php-fpm.conf
rm -f php-fpm.d/*.conf php-fpm.conf.default php-fpm.d/www.conf.default

cat <<'INNER_EOF' > php-fpm.d/00-global.conf
[global]
error_log = /proc/self/fd/2

; https://github.com/docker-library/php/pull/725#issuecomment-443540114
log_limit = 8192

; Keep PHP-FPM in the foreground as PID 1 in the container.
daemonize = no
INNER_EOF

cat <<'INNER_EOF' > php-fpm.d/10-www.conf
[www]
user = www-data
group = www-data

; Listen on all container interfaces. Docker controls whether port 9000 is published externally.
listen = 9000
listen.backlog = 1024

pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 5

; Recycle workers periodically to release memory retained by PHP extensions or third-party libraries.
pm.max_requests = 500

; PHP has max_execution_time, but this provides a PHP-FPM-level hard limit for stuck requests.
request_terminate_timeout = 60s

; Dump a PHP backtrace for requests taking longer than 5 seconds.
request_slowlog_timeout = 5s
request_slowlog_trace_depth = 20
slowlog = /proc/self/fd/2

; php-fpm closes STDOUT on startup, so sending logs to /proc/self/fd/1 does not work.
; https://bugs.php.net/bug.php?id=73886
access.log = /proc/self/fd/2

; Preserve Docker environment variables for PHP workers.
clear_env = no

; Ensure worker stdout and stderr are sent to the main error log.
catch_workers_output = yes
decorate_workers_output = no

; Only allow PHP scripts to be executed through this FPM pool.
security.limit_extensions = .php
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/00-fpm.ini"
; https://github.com/docker-library/php/issues/878#issuecomment-938595965
fastcgi.logging = Off
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/10-opcache.ini"
[opcache]
opcache.enable = 1
opcache.enable_cli = 0

; Shared OPcache memory used by all PHP-FPM workers.
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000

; Containers are immutable: PHP-FPM is restarted when a new image is deployed.
; opcache.validate_timestamps = 0

; Check mounted PHP files for changes and revalidate cached scripts every second.
opcache.validate_timestamps = 1
opcache.revalidate_freq = 1

; Preserve PHPDoc comments used by frameworks, annotations and reflection.
opcache.save_comments = 1

; JIT is generally not useful for typical PHP web workloads.
opcache.jit = off
opcache.jit_buffer_size = 0
INNER_EOF

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/20-security.ini"
cgi.fix_pathinfo = 0
expose_php = Off
display_errors = Off
display_startup_errors = Off
allow_url_fopen = Off
allow_url_include = Off

; Session security
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

cat <<'INNER_EOF' > "${PHP_INI_DIR}/conf.d/41-sodium.ini"
extension = sodium.so
INNER_EOF

EOF

RUN php --version \
    && php --ini \
    && php -m

FROM alpine:3.24

WORKDIR /var/www/html

ENV TZ=Asia/Shanghai

COPY --from=builder /usr/local/ /usr/local/

RUN set -eux \
    && adduser -u 82 -D -S -G www-data www-data \
    && chown www-data:www-data /var/www/html \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --no-cache $runDeps tzdata \
    && php-fpm -t

EXPOSE 9000
STOPSIGNAL SIGQUIT
ENTRYPOINT ["docker-php-entrypoint"]
CMD ["php-fpm"]
