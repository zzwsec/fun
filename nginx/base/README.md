# NGINX 多架构构建

在仓库根目录执行构建和推送命令。

```sh
# 首次使用时创建并切换到支持多平台的 builder
# docker buildx create --name multiarch --driver docker-container --use --bootstrap

NGINX_VERSION=1.31.9

docker buildx build --platform linux/amd64,linux/arm64 --target static \
    --build-arg NGINX_VERSION="$NGINX_VERSION" \
    --tag "zzwsec/nginx:$NGINX_VERSION" --tag zzwsec/nginx:latest \
    --push nginx/base

docker buildx build --platform linux/amd64,linux/arm64 --target dynamic \
    --build-arg NGINX_VERSION="$NGINX_VERSION" \
    --tag "zzwsec/nginx:${NGINX_VERSION}-dynamic" \
    --push nginx/base
```
