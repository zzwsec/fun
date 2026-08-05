# OpenResty Alpine 镜像

使用 Alpine 多阶段构建 OpenResty，并单独编译 OpenSSL 3 和 PCRE2。运行时镜像包含 OpenResty 及其运行依赖。

## 构建

在仓库根目录执行：

```shell
docker build \
    --tag zzwsec/openresty:1.31.1-alpine \
    openresty/static
```

主要构建参数：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `RESTY_IMAGE_TAG` | `alpine:3.24.1` | 构建阶段和运行阶段的 Alpine 基础镜像 |
| `RESTY_VERSION` | `1.31.1.1` | OpenResty 源码版本 |
| `RESTY_OPENSSL_VERSION` | `3.5.7` | OpenSSL 源码版本 |
| `RESTY_OPENSSL_PATCH_VERSION` | `3.5.5` | OpenResty OpenSSL 补丁版本 |
| `RESTY_PCRE_VERSION` | `10.47` | PCRE2 源码版本 |
| `RESTY_J` | 空 | 并行编译任务数；空值使用 `nproc` |

指定并行编译任务数：

```shell
docker build \
    --build-arg RESTY_J=8 \
    --tag zzwsec/openresty:1.31.1-alpine \
    openresty/static
```

检查镜像：

```shell
docker run --rm zzwsec/openresty:1.31.1-alpine openresty -V
docker run --rm zzwsec/openresty:1.31.1-alpine openresty -t
```

## 静态模块

以下第三方模块静态链接至 NGINX 主程序：

- `ngx_brotli`：Brotli 压缩
- `zstd-nginx-module`：Zstandard 压缩
- `nginx-acme`：ACME 证书管理
