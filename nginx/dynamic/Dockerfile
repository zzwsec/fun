ARG NGINX_VERSION

FROM alpine AS builder
ARG NGINX_VERSION

RUN apk add --no-cache \
    build-base pcre2-dev zlib-dev openssl-dev linux-headers git curl \
    gd-dev libmaxminddb-dev libxml2-dev libxslt-dev

WORKDIR /usr/src
RUN curl -fsSL https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz | tar xzf - 

WORKDIR /usr/src/nginx-${NGINX_VERSION}

RUN git clone --depth=1 https://github.com/nginx/njs.git \
    && git clone --depth=1 https://github.com/leev/ngx_http_geoip2_module.git \
    && mkdir -p /tmp/modules-export

RUN ./configure \
    --with-compat \
    --with-stream \
    --with-http_image_filter_module=dynamic \
    --add-dynamic-module=njs/nginx \
    --add-dynamic-module=ngx_http_geoip2_module \
    && make modules \
    && strip objs/*.so \
    && cp objs/*.so /tmp/modules-export/

FROM zzwsec/nginx:${NGINX_VERSION}

RUN apk add --no-cache gd libmaxminddb libxml2 libxslt
COPY --from=builder /tmp/modules-export/ /usr/lib/nginx/modules/