# OpenResty LuaRocks 镜像

基于 `zzwsec/openresty:1.31.1.1` 构建，在独立阶段编译 LuaRocks 和原生 Lua 模块，最终只把已安装的 rocks 及其运行依赖复制到 OpenResty 运行时镜像。

## 预装模块

| Rock | Lua 加载名称 | 说明 |
|---|---|---|
| `luaossl` | `openssl` | 基于镜像私有 OpenSSL 3 的密码学和 TLS 接口 |
| `lapis` | `lapis` | OpenResty Web 框架 |
| `lua-resty-mlcache` | `resty.mlcache` | 基于 worker LRU 和共享字典的多级缓存 |
| `lua-resty-jit-uuid` | `resty.jit-uuid` | LuaJIT/ngx_lua UUID 生成器 |
| `luasql-mysql` | `luasql.mysql` | MySQL/MariaDB LuaSQL 驱动 |

LuaRocks 会自动安装这些 rocks 的传递依赖。原生模块安装在 `/usr/lib/openresty/luajit/lib/lua/5.1/`，Lua 文件安装在 `/usr/lib/openresty/luajit/share/lua/5.1/`。最终镜像额外安装 `mariadb-connector-c`，供 `luasql-mysql` 运行时使用。

## 构建

在仓库根目录执行：

```shell
docker build -t zzwsec/openresty:1.31.1.1-luarocks openresty/luarocks
```

构建前应先构建或拉取 `zzwsec/openresty:1.31.1.1`；同一基础镜像同时用于 rocks 编译阶段和最终运行阶段。

主要构建参数：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `OPENRESTY_IMAGE` | `zzwsec/openresty:1.31.1.1` | 构建阶段和最终运行阶段的基础镜像 |
| `RESTY_VERSION` | `1.31.1.1` | 用于取得匹配 LuaJIT 头文件的 OpenResty 源码版本 |
| `RESTY_OPENSSL_VERSION` | `3.5.7` | 与基础镜像保持一致的 OpenSSL 源码版本 |
| `RESTY_OPENSSL_PATCH_VERSION` | `3.5.5` | 应用于 OpenSSL 的 OpenResty 补丁版本 |
| `RESTY_LUAROCKS_VERSION` | `3.13.0` | 构建阶段使用的 LuaRocks 版本 |
| `RESTY_J` | 空 | OpenSSL 并行编译任务数；空值使用 `nproc` |
| `RESTY_OPENSSL_URL_BASE` | OpenSSL GitHub Release 地址 | OpenSSL 源码下载地址前缀 |

`RESTY_OPENSSL_BUILD_OPTIONS` 可通过 `--build-arg` 整体覆盖。`luaossl` 使用与基础镜像相同版本、相同补丁和相同功能参数的私有 OpenSSL 进行编译，并通过 RPATH 在运行时加载 `/usr/lib/openresty/openssl3/lib` 下的共享库。

## 验证

纯 Lua 或不依赖 `ngx` 请求上下文的模块可以直接通过 LuaJIT 检查：

```shell
docker run --rm \
    zzwsec/openresty:1.31.1.1-luarocks \
    luajit -e '
        assert(require("openssl"))
        assert(require("lapis"))
        local uuid = assert(require("resty.jit-uuid"))
        uuid.seed()
        assert(uuid.is_valid(uuid.generate_v4()))
        local mysql = assert(require("luasql.mysql"))
        assert(mysql.mysql()):close()
        print("rocks ok")
    '
```

`resty.mlcache` 依赖 `ngx` API 和 `lua_shared_dict`，应在 OpenResty 请求上下文中加载和使用，不能用裸 `luajit` 直接验证。

检查基础配置：

```shell
docker run --rm \
    zzwsec/openresty:1.31.1.1-luarocks \
    openresty -t
```

## 版本兼容

`OPENRESTY_IMAGE`、`RESTY_VERSION`、`RESTY_OPENSSL_VERSION` 和 `RESTY_OPENSSL_PATCH_VERSION` 必须保持同步。升级基础镜像后应重新构建本镜像，避免原生 Lua 模块使用与运行时不一致的 LuaJIT 头文件或 OpenSSL ABI。

该镜像只提供预装 rocks，不提供运行时执行 `luarocks install` 的能力。如需增加或升级模块，应修改 Dockerfile 后重新构建镜像。
