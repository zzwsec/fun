# syntax=docker/dockerfile:1

ARG ALPINE_IMAGE="alpine:3.24"
ARG NGINX_VERSION="1.31.5"
ARG NGINX_IMAGE="zzwsec/nginx:${NGINX_VERSION}"

FROM ${ALPINE_IMAGE} AS builder
ARG NGINX_VERSION

RUN apk add --no-cache \
    build-base curl gd-dev git libmaxminddb-dev libxml2-dev \
    libxslt-dev openssl-dev pcre2-dev zlib-dev

WORKDIR /usr/src
RUN curl -fsSL https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz | tar xzf - \
    && git clone --depth=1 https://github.com/nginx/njs.git nginx-${NGINX_VERSION}/njs \
    && git clone --depth=1 https://github.com/leev/ngx_http_geoip2_module.git nginx-${NGINX_VERSION}/ngx_http_geoip2_module

WORKDIR /usr/src/nginx-${NGINX_VERSION}
RUN ./configure \
    --with-compat \
    --with-stream \
    --with-http_image_filter_module=dynamic \
    --add-dynamic-module=njs/nginx \
    --add-dynamic-module=ngx_http_geoip2_module \
    && make modules \
    && strip objs/*.so \
    && mkdir -p /usr/lib/nginx/modules \
    && cp objs/*.so /usr/lib/nginx/modules/

FROM ${NGINX_IMAGE}

RUN apk add --no-cache libgd libmaxminddb-libs libxml2 libxslt
COPY --from=builder /usr/lib/nginx/modules/ /usr/lib/nginx/modules/
