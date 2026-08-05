# syntax=docker/dockerfile:1

# OpenResty base image with statically linked third-party modules.

ARG RESTY_IMAGE_TAG="alpine:3.24.1"

FROM ${RESTY_IMAGE_TAG} AS builder

ARG RESTY_VERSION="1.31.1.1"
ARG RESTY_J=""

ARG NGX_BROTLI_REPO="https://github.com/google/ngx_brotli.git"
ARG NGX_ZSTD_REPO="https://github.com/tokers/zstd-nginx-module.git"
ARG NGX_ACME_REPO="https://github.com/nginx/nginx-acme.git"

# https://github.com/openresty/openresty-packaging/blob/master/alpine/openresty-openssl3/APKBUILD
ARG RESTY_OPENSSL_VERSION="3.5.7"
ARG RESTY_OPENSSL_PATCH_VERSION="3.5.5"
ARG RESTY_OPENSSL_URL_BASE="https://github.com/openssl/openssl/releases/download/openssl-${RESTY_OPENSSL_VERSION}"
ARG RESTY_OPENSSL_BUILD_OPTIONS="\
    enable-camellia enable-cms enable-fips enable-ktls enable-md2 enable-rc5 \
    enable-rfc3779 enable-seed enable-ssl3 enable-ssl3-method \
    enable-weak-ssl-ciphers \
    "

# https://github.com/openresty/openresty-packaging/blob/master/alpine/openresty-pcre2/APKBUILD
ARG RESTY_PCRE_VERSION="10.47"
ARG RESTY_PCRE_BUILD_OPTIONS="\
    --disable-bsr-anycrlf --disable-coverage --disable-ebcdic --disable-fuzz-support \
    --disable-jit-sealloc --disable-never-backslash-C \
    --disable-pcre2grep-libbz2 --disable-pcre2grep-libz --disable-pcre2test-libedit \
    --disable-rebuild-chartables --disable-silent-rules --disable-static --disable-valgrind \
    --enable-jit --enable-newline-is-lf \
    --enable-pcre2-8 --enable-pcre2-16 --enable-pcre2-32 \
    --enable-pcre2grep-callout --enable-pcre2grep-callout-fork --enable-pcre2grep-jit \
    --enable-percent-zt --enable-shared --enable-unicode \
    "

# https://github.com/openresty/openresty-packaging/blob/master/alpine/openresty/APKBUILD
ARG RESTY_CONFIG_OPTIONS="\
    --with-compat --with-threads \
    --with-http_addition_module --with-http_auth_request_module --with-http_flv_module \
    --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module \
    --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module \
    --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module \
    --with-http_sub_module --with-http_v2_module --with-http_v3_module \
    --with-stream --with-stream_ssl_module --with-stream_ssl_preread_module \
    --without-http_rds_csv_module --without-http_rds_json_module --without-lua_rds_parser \
    "
ARG RESTY_CONFIG_OPTIONS_MORE="\
    --add-module=ngx_brotli \
    --add-module=ngx_zstd \
    --add-module=ngx_http_acme_module \
    "
ARG RESTY_LUAJIT_OPTIONS="--with-luajit-xcflags='-DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT'"
ARG RESTY_PCRE_OPTIONS="--with-pcre-jit"
ARG _RESTY_CONFIG_DEPS="\
    --with-cc-opt='-DNGX_LUA_ABORT_AT_PANIC -I/usr/local/openresty/pcre2/include -I/usr/local/openresty/openssl3/include' \
    --with-ld-opt='-L/usr/local/openresty/pcre2/lib -L/usr/local/openresty/openssl3/lib -Wl,-rpath,/usr/local/openresty/pcre2/lib:/usr/local/openresty/openssl3/lib' \
    --with-pcre \
    "

RUN apk add --no-cache \
    build-base binutils cargo clang clang-dev cmake coreutils curl git \
    linux-headers make perl-dev pkgconf readline-dev \
    zlib-dev zstd-dev zstd-static \
    libgcc tzdata zlib

RUN curl -sSfL "${RESTY_OPENSSL_URL_BASE}/openssl-${RESTY_OPENSSL_VERSION}.tar.gz" | tar zxf - -C /tmp/ \
    && curl -sSfL "https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-${RESTY_OPENSSL_PATCH_VERSION}-sess_set_get_cb_yield.patch" | patch -d "/tmp/openssl-${RESTY_OPENSSL_VERSION}" -p1 \
    && curl -sSfL "https://github.com/PCRE2Project/pcre2/releases/download/pcre2-${RESTY_PCRE_VERSION}/pcre2-${RESTY_PCRE_VERSION}.tar.gz" | tar zxf - -C /tmp/ \
    && curl -sSfL "https://openresty.org/download/openresty-${RESTY_VERSION}.tar.gz" | tar zxf - -C /tmp/ \
    && git clone --depth=1 --recurse-submodules --shallow-submodules "${NGX_BROTLI_REPO}" "/tmp/openresty-${RESTY_VERSION}/ngx_brotli" \
    && git clone --depth=1 "${NGX_ZSTD_REPO}" "/tmp/openresty-${RESTY_VERSION}/ngx_zstd" \
    && git clone --depth=1 "${NGX_ACME_REPO}" "/tmp/openresty-${RESTY_VERSION}/ngx_http_acme_module"

# https://github.com/google/ngx_brotli#statically-compiled
WORKDIR /tmp/openresty-${RESTY_VERSION}
RUN RESTY_BUILD_JOBS="${RESTY_J:-$(nproc)}" \
    && cmake \
        -S ngx_brotli/deps/brotli \
        -B ngx_brotli/deps/brotli/out \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_SHARED_LIBS=OFF \
        -DBROTLI_DISABLE_TESTS=ON \
    && cmake --build ngx_brotli/deps/brotli/out \
        --config Release \
        --target brotlienc \
        --parallel "${RESTY_BUILD_JOBS}"

WORKDIR /tmp/openssl-${RESTY_OPENSSL_VERSION}
RUN RESTY_BUILD_JOBS="${RESTY_J:-$(nproc)}" \
    && ./config \
        shared zlib -g \
        --prefix=/usr/local/openresty/openssl3 \
        --libdir=lib \
        -Wl,-rpath,/usr/local/openresty/openssl3/lib \
        ${RESTY_OPENSSL_BUILD_OPTIONS} \
    && make -j"${RESTY_BUILD_JOBS}" \
    && make -j"${RESTY_BUILD_JOBS}" install_sw

WORKDIR /tmp/pcre2-${RESTY_PCRE_VERSION}
RUN RESTY_BUILD_JOBS="${RESTY_J:-$(nproc)}" \
    && CFLAGS="-g -O3" ./configure \
        --prefix=/usr/local/openresty/pcre2 \
        --libdir=/usr/local/openresty/pcre2/lib \
        ${RESTY_PCRE_BUILD_OPTIONS} \
    && make -j"${RESTY_BUILD_JOBS}" \
    && make -j"${RESTY_BUILD_JOBS}" install

WORKDIR /tmp/openresty-${RESTY_VERSION}
RUN RESTY_BUILD_JOBS="${RESTY_J:-$(nproc)}" \
    && export \
        CARGO_BUILD_JOBS="${RESTY_BUILD_JOBS}" \
        OPENSSL_DIR=/usr/local/openresty/openssl3 \
        OPENSSL_INCLUDE_DIR=/usr/local/openresty/openssl3/include \
        OPENSSL_LIB_DIR=/usr/local/openresty/openssl3/lib \
        PKG_CONFIG_PATH=/usr/local/openresty/openssl3/lib/pkgconfig \
        ZSTD_INC=/usr/include \
        ZSTD_LIB=/usr/lib \
    && eval ./configure -j"${RESTY_BUILD_JOBS}" ${_RESTY_CONFIG_DEPS} ${RESTY_CONFIG_OPTIONS} ${RESTY_CONFIG_OPTIONS_MORE} ${RESTY_LUAJIT_OPTIONS} ${RESTY_PCRE_OPTIONS} \
    && sed -i \
        -e "s|--add-module=${PWD}/ngx_brotli|--add-module=ngx_brotli|g" \
        -e "s|--add-module=${PWD}/ngx_zstd|--add-module=ngx_zstd|g" \
        -e "s|--add-module=${PWD}/ngx_http_acme_module|--add-module=ngx_http_acme_module|g" \
        build/nginx-*/objs/ngx_auto_config.h \
    && make -j"${RESTY_BUILD_JOBS}" \
    && make -j"${RESTY_BUILD_JOBS}" install

RUN rm -Rf \
        /usr/local/openresty/openssl3/bin/c_rehash \
        /usr/local/openresty/openssl3/lib/*.a \
        /usr/local/openresty/openssl3/include \
    && find /usr/local/openresty/openssl3 -type f -perm -u+x -exec strip --strip-unneeded '{}' \; \
    && rm -Rf \
        /usr/local/openresty/pcre2/bin \
        /usr/local/openresty/pcre2/share \
        /usr/local/openresty/pcre2/lib/*.a \
        /usr/local/openresty/pcre2/lib/*.la \
    && find /usr/local/openresty/pcre2 -type f \( -name '*.so' -o -name '*.so.*' \) -exec strip --strip-unneeded '{}' \; \
    && rm -Rf \
        /usr/local/openresty/luajit/lib/*.a \
        /usr/local/openresty/luajit/share/man \
    && find /usr/local/openresty/luajit -type f -perm -u+x -exec strip --strip-unneeded '{}' \; \
    && find /usr/local/openresty/nginx -type f -perm -u+x -exec strip --strip-unneeded '{}' \;

RUN mkdir -p /etc/nginx/conf.d \
    && curl -fsSL \
        -o /usr/local/openresty/nginx/conf/nginx.conf \
        "https://github.com/openresty/docker-openresty/raw/refs/tags/${RESTY_VERSION}-0/nginx.conf" \
    && curl -fsSL \
        -o /etc/nginx/conf.d/default.conf \
        "https://github.com/openresty/docker-openresty/raw/refs/tags/${RESTY_VERSION}-0/nginx.vh.default.conf"

FROM ${RESTY_IMAGE_TAG} AS runtime

ENV PATH="/usr/local/openresty/bin:/usr/local/openresty/nginx/sbin:/usr/local/openresty/luajit/bin:/usr/local/openresty/openssl3/bin:${PATH}" \
    RESTY_VERSION="1.31.1.1" \
    RESTY_PCRE_VERSION="10.47" \
    RESTY_OPENSSL_VERSION="3.5.7" \
    RESTY_OPENSSL_PATCH_VERSION="3.5.5"

COPY --from=builder /usr/local/openresty/ /usr/local/openresty/
COPY --from=builder /etc/nginx/ /etc/nginx/

RUN apk add --no-cache libgcc tzdata zlib \
    && mkdir -p /var/run/openresty \
    && ln -sf /dev/stdout /usr/local/openresty/nginx/logs/access.log \
    && ln -sf /dev/stderr /usr/local/openresty/nginx/logs/error.log

CMD ["/usr/local/openresty/bin/openresty", "-g", "daemon off;"]

# https://github.com/openresty/docker-openresty/blob/master/README.md#tips--pitfalls
STOPSIGNAL SIGQUIT
