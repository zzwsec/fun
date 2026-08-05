# syntax=docker/dockerfile:1
FROM php:8.1.34-fpm-alpine3.21

ENV TZ=UTC

# wget -qO /usr/bin/install-php-extensions "https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions"
RUN --mount=type=bind,from=mlocati/php-extension-installer:2,source=/usr/bin/install-php-extensions,target=/usr/local/bin/install-php-extensions \
    apk add --no-cache --no-progress tzdata \
    && echo 'date.timezone = ${TZ}' > "$PHP_INI_DIR/conf.d/timezone.ini" \
    && install-php-extensions \
        bcmath bz2 calendar exif gd gettext gmp ldap mysqli odbc \
        pdo_mysql soap shmop sysvmsg sysvsem sysvshm xmlrpc xsl