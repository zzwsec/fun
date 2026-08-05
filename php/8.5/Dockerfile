FROM alpine:3.24 AS builder

ARG PHP_VERSION=8.5.8
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
        --enable-ftp \
        --enable-gd \
        --enable-intl \
        --enable-mbstring \
        --enable-mysqlnd \
        --enable-option-checking=fatal \
        --enable-pcntl \
        --enable-shmop \
        --enable-soap \
        --enable-sysvmsg \
        --enable-sysvsem \
        --enable-sysvshm \
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
    && git clone --depth 1 https://github.com/phpredis/phpredis.git /usr/src/phpredis \
    && cd /usr/src/phpredis \
    && phpize \
    && ./configure --with-php-config=/usr/local/bin/php-config \
    && make -j$(nproc) \
    && make install

RUN set -eux \
    && find /usr/local/ -type f -perm '/0111' -exec sh -c ' \
        for f; do \
            if file "$f" | grep -q "ELF"; then \
                strip --strip-all "$f"; \
            fi \
        done \
        ' sh {} + \
    && mkdir -p "${PHP_INI_DIR}/conf.d" \
    && cp -f /usr/php-${PHP_VERSION}/php.ini-* "${PHP_INI_DIR}/" \
    && curl -fsSL https://github.com/docker-library/php/archive/refs/heads/master.tar.gz | tar zxf - -C /usr/local/bin/ php-master/docker-php-entrypoint php-master/docker-php-ext-configure php-master/docker-php-ext-enable php-master/docker-php-ext-install --strip-components=1 \
    && chmod +x /usr/local/bin/docker-php-*

WORKDIR /usr/local/etc

RUN <<EOF
set -eux

sed 's|=NONE/|=|g' php-fpm.conf.default > php-fpm.conf
cp -f php-fpm.d/www.conf.default php-fpm.d/www.conf

cat <<\INNER_EOF > php-fpm.d/docker.conf
[global]
error_log = /proc/self/fd/2

; https://github.com/docker-library/php/pull/725#issuecomment-443540114
log_limit = 8192

[www]
; php-fpm closes STDOUT on startup, so sending logs to /proc/self/fd/1 does not work.
; https://bugs.php.net/bug.php?id=73886
access.log = /proc/self/fd/2

clear_env = no

; Ensure worker stdout and stderr are sent to the main error log.
catch_workers_output = yes
decorate_workers_output = no
INNER_EOF

cat <<\INNER_EOF > php-fpm.d/zz-docker.conf
[global]
daemonize = no

[www]
listen = 9000
INNER_EOF

echo -e '; https://github.com/docker-library/php/issues/878#issuecomment-938595965\nfastcgi.logging = Off' > "${PHP_INI_DIR}/conf.d/docker-fpm.ini"
echo 'date.timezone = ${TZ}' > "${PHP_INI_DIR}/conf.d/timezone.ini"
echo "extension=sodium.so" > "${PHP_INI_DIR}/conf.d/docker-php-ext-sodium.ini"
echo "extension=redis.so" > "${PHP_INI_DIR}/conf.d/docker-php-ext-redis.ini"
echo -e "cgi.fix_pathinfo = 0\nexpose_php = Off\ndisplay_errors = Off\ndisplay_startup_errors = Off\nallow_url_fopen = Off\nallow_url_include = Off" > "${PHP_INI_DIR}/conf.d/zz-security.ini"

EOF

FROM scratch

COPY --from=alpine:3.24 / /
COPY --from=builder /usr/local/ /usr/local/

ENV TZ=UTC

RUN set -eux \
    && adduser -u 82 -D -S -G www-data www-data \
    && mkdir -p /var/www/html \
    && chown www-data:www-data /var/www/html \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --no-cache $runDeps tzdata

WORKDIR /var/www/html

EXPOSE 9000
STOPSIGNAL SIGQUIT
ENTRYPOINT ["docker-php-entrypoint"]
CMD ["php-fpm"]