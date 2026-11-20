# syntax=docker/dockerfile:1
FROM php:8.5.13-fpm-alpine3.24

ARG PHPREDIS_VERSION=6.3.0
ENV TZ=Asia/Shanghai

RUN --mount=type=bind,from=mlocati/php-extension-installer:2,source=/usr/bin/install-php-extensions,target=/usr/local/bin/install-php-extensions \
    set -eux \
    && apk add --no-cache tzdata \
    && printf '%s\n' 'date.timezone = ${TZ}' > "${PHP_INI_DIR}/conf.d/timezone.ini" \
    && install-php-extensions \
        bcmath bz2 calendar exif gd gettext gmp intl ldap mysqli odbc \
        pcntl pdo_mysql shmop soap xsl zip "redis-${PHPREDIS_VERSION}" \
    && php-fpm -t
