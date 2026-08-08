# 从 Dify 官方源码构建 `/nuaai01` 子路径版本

本文从 Dify 官方 GitHub 仓库的固定提交开始，不依赖任何已经修改好的个人分支，也不依赖已有前端镜像。

最终访问链路：

```text
外部用户
  ↓
https://生产域名/nuaai01
  ↓
生产入口服务器上的原生 Nginx
  ↓ 保留 /nuaai01
http://Dify服务器内网IP:8080
  ↓
Podman Compose 内置 Nginx
  ├─ API、文件、插件、Socket.IO：删除 /nuaai01 后转发
  └─ Next.js Web：保留 /nuaai01 后转发
```

---

## 1. 固定官方源码版本

后端镜像固定为：

```text
langgenius/dify-api:1.16.1
```

与该镜像对齐的官方源码提交：

```text
6f8ed69ee15f9a2e7189ca066275e973d091d1e9
```

官方仓库：

```text
https://github.com/langgenius/dify.git
```

克隆并创建改造分支：

```bash
cd /home/ubuntu

git clone https://github.com/langgenius/dify.git dify
cd /home/ubuntu/dify

git checkout 6f8ed69ee15f9a2e7189ca066275e973d091d1e9
git switch -c release-1.16.1-nuaai01

git status
git rev-parse HEAD
```

`git rev-parse HEAD` 必须输出：

```text
6f8ed69ee15f9a2e7189ca066275e973d091d1e9
```

### 为什么必须固定这个提交

前端源码与后端 API 镜像必须来自同一版本。使用较新的 `main` 前端配合旧的 `dify-api:1.16.1`，前端可能请求后端不存在的接口，例如：

```text
/console/api/workspaces/current/summary
/console/api/workspaces/current/model-providers/summary
```

结果是进入工作区或工作流页面时出现 404。这个问题不能通过 Nginx rewrite 解决，唯一正确做法是让前端源码和后端镜像版本对齐。

副作用说明：固定提交意味着不会自动获得官方后续功能和安全修复。升级 Dify 时必须重新寻找与新后端镜像对应的源码提交，并重新迁移本文修改。

---

## 2. 理解哪些内容 Next.js 已经支持

官方 `web/Dockerfile` 已经包含：

```dockerfile
ARG NEXT_PUBLIC_BASE_PATH=""
ENV NEXT_PUBLIC_BASE_PATH="$NEXT_PUBLIC_BASE_PATH"
```

官方 `web/next.config.ts` 已经包含：

```typescript
basePath: env.NEXT_PUBLIC_BASE_PATH,
```

因此以下内容不需要自行重写：

- Next.js 页面路由；
- `next/link`；
- Dify 封装的 Next Router；
- `/_next` 静态资源；
- 已经使用 `basePath` 的图片、字体和 Monaco 资源。

但是 `NEXT_PUBLIC_BASE_PATH` 是构建期变量，官方预构建的 `langgenius/dify-web:1.16.1` 不能在容器启动时完整切换到 `/nuaai01`。必须重新构建 Web 镜像。

副作用说明：使用 `/nuaai01` 构建出来的前端镜像只能作为 `/nuaai01` 版本使用。以后改成 `/dify` 或根路径时必须重新构建另一个镜像。

---

## 3. 修改前端 WebSocket 路径

文件：

```text
web/app/components/workflow/collaboration/core/websocket-manager.ts
```

在 import 区域增加：

```typescript
import { basePath } from '@/utils/var'
```

找到：

```typescript
path: '/socket.io',
```

改为：

```typescript
path: `${basePath}/socket.io`,
```

这是子路径部署的必要修改。Socket.IO 客户端的 `path` 不受 Next.js `basePath` 自动处理。如果不改，浏览器会请求：

```text
/socket.io
```

正确请求必须是：

```text
/nuaai01/socket.io
```

`NEXT_PUBLIC_SOCKET_URL` 只填写 Origin：

```env
NEXT_PUBLIC_SOCKET_URL=wss://生产域名
```

不能填写：

```env
NEXT_PUBLIC_SOCKET_URL=wss://生产域名/nuaai01/socket.io
```

否则 URL 和 Socket.IO path 会重复拼接。

---

## 4. 修改 Device Flow 根路径请求

### 4.1 Device Flow API

文件：

```text
web/service/device-flow.ts
```

增加：

```typescript
import { basePath } from '@/utils/var'
```

把：

```typescript
const DEVICE_BASE = '/openapi/v1/oauth/device'
```

改成：

```typescript
const DEVICE_BASE = `${basePath}/openapi/v1/oauth/device`
```

这是必要修改，因为这里直接调用浏览器 `fetch`，不会由 Next Router 自动增加 `basePath`。

### 4.2 Device Flow SSO

文件：

```text
web/app/device/components/chooser.tsx
```

增加：

```typescript
import { basePath } from '@/utils/var'
```

把：

```typescript
window.location.href = `/openapi/v1/oauth/device/sso-initiate?user_code=${encodeURIComponent(userCode)}`
```

改成：

```typescript
window.location.href = `${basePath}/openapi/v1/oauth/device/sso-initiate?user_code=${encodeURIComponent(userCode)}`
```

这是必要修改，因为 `window.location.href` 不会自动处理 Next.js `basePath`。

文件中的：

```typescript
router.push('/signin')
```

不要额外拼接 `basePath`。这里使用的是项目封装的 Next Router，重复拼接可能产生 `/nuaai01/nuaai01/signin`。

---

## 5. 修改浏览器原生页面跳转

### 5.1 登录跳转

文件：

```text
web/service/fetch.ts
```

增加：

```typescript
import { basePath } from '@/utils/var'
```

把：

```typescript
globalThis.location.href = `${globalThis.location.origin}/signin`
```

改成：

```typescript
globalThis.location.href = `${globalThis.location.origin}${basePath}/signin`
```

### 5.2 Account 命令

文件：

```text
web/app/components/goto-anything/actions/commands/account.tsx
```

增加：

```typescript
import { basePath } from '@/utils/var'
```

将文件中的两处：

```typescript
window.location.href = '/account'
```

都改成：

```typescript
window.location.href = `${basePath}/account`
```

### 5.3 Goto Anything 导航

文件：

```text
web/app/components/goto-anything/actions/commands/go.tsx
```

增加：

```typescript
import { basePath } from '@/utils/var'
```

把：

```typescript
if (args?.path) window.location.href = args.path
```

改成：

```typescript
if (args?.path) window.location.href = `${basePath}${args.path}`
```

以上三处都是必要修改，因为它们使用浏览器原生跳转。不要全局搜索所有 `router.push('/...')` 并机械拼接 `basePath`，Next Router 通常已经自动处理，机械修改会造成双前缀。

---

## 6. 修改 Dify 内置 Nginx 路由

文件：

```text
docker/nginx/conf.d/default.conf.template
```

### 6.1 API 路由

把以下 location：

```nginx
/console/api
/api
/v1
/openapi
/files
/mcp
/triggers
```

全部增加 `${DIFY_BASE_PATH}`，并在转发前删除前缀。

以 Console API 为例，原配置：

```nginx
location /console/api {
  proxy_pass http://api:5001;
  include proxy.conf;
}
```

改为：

```nginx
location ${DIFY_BASE_PATH}/console/api {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}
```

其他 API 路由采用相同模式：

```nginx
location ${DIFY_BASE_PATH}/api {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}

location ${DIFY_BASE_PATH}/v1 {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}

location ${DIFY_BASE_PATH}/openapi {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}

location ${DIFY_BASE_PATH}/files {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}

location ${DIFY_BASE_PATH}/mcp {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}

location ${DIFY_BASE_PATH}/triggers {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://api:5001;
  include proxy.conf;
}
```

删除前缀是必要的，因为 Flask API 仍然注册在 `/console/api`、`/api`、`/v1` 等根路径下。

### 6.2 WebSocket 路由

原配置包含 Docker 专用 DNS：

```nginx
location /socket.io/ {
  resolver 127.0.0.11 valid=30s ipv6=off;
  set $socket_io_upstream ${NGINX_SOCKET_IO_UPSTREAM};
  proxy_pass http://$socket_io_upstream;
  include proxy.conf;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_cache_bypass $http_upgrade;
}
```

改为：

```nginx
location ${DIFY_BASE_PATH}/socket.io/ {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
  proxy_pass http://${NGINX_SOCKET_IO_UPSTREAM};
  include proxy.conf;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_cache_bypass $http_upgrade;
}
```

其中增加 `${DIFY_BASE_PATH}` 和 rewrite 是子路径必需修改；删除 `resolver 127.0.0.11` 是 Podman 兼容修改，因为 `127.0.0.11` 是 Docker 内置 DNS，不是 Podman 通用地址。

副作用：直接写上游服务名时，Nginx 启动阶段必须能够解析 `api_websocket`。因此部署时必须启用 `collaboration` Profile，并保证 `api_websocket` 被创建。若不启用协作服务，内置 Nginx 可能因无法解析上游而反复重启。

### 6.3 插件路由

把：

```nginx
location /e/ {
```

改为：

```nginx
location ${DIFY_BASE_PATH}/e/ {
  rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
```

保留原有：

```nginx
proxy_pass http://plugin_daemon:5002;
proxy_set_header Dify-Hook-Url $scheme://$host$request_uri;
include proxy.conf;
```

不要在不知道生产入口 Nginx 转发头策略时修改 `proxy.conf.template` 或全局改写 `X-Forwarded-Proto`。外层 Nginx 的域名、证书、TLS 和可信代理策略属于生产入口配置，不应通过测试环境推断后强行写入 Dify 内置 Nginx。

### 6.4 Web 路由

把：

```nginx
location /explore {
```

改为：

```nginx
location ${DIFY_BASE_PATH}/explore {
```

这里不要 rewrite，因为 Next.js 必须看到 `/nuaai01/explore`。

把根 Web 路由改成：

```nginx
${BASE_PATH_EXACT_LOCATION}

location ${DIFY_BASE_PATH}/ {
  proxy_pass http://web:3000;
  include proxy.conf;
}
```

这里同样不要删除 `/nuaai01`。使用 `NEXT_PUBLIC_BASE_PATH=/nuaai01` 构建的 Next.js 服务要求请求 URI 保留前缀。

---

## 7. 为内置 Nginx 增加 DIFY_BASE_PATH 校验

文件：

```text
docker/nginx/docker-entrypoint.sh
```

紧跟在：

```bash
HTTPS_CONFIG=''
```

后面增加：

```bash
DIFY_BASE_PATH=${DIFY_BASE_PATH:-}
case "${DIFY_BASE_PATH}" in
    "")
        BASE_PATH_EXACT_LOCATION=''
        ;;
    /*/)
        echo "DIFY_BASE_PATH must not end with a slash: ${DIFY_BASE_PATH}" >&2
        exit 1
        ;;
    /*)
        BASE_PATH_EXACT_LOCATION="location = ${DIFY_BASE_PATH} { proxy_pass http://web:3000; include proxy.conf; }"
        ;;
    *)
        echo "DIFY_BASE_PATH must be empty or start with a slash: ${DIFY_BASE_PATH}" >&2
        exit 1
        ;;
esac
export DIFY_BASE_PATH
export BASE_PATH_EXACT_LOCATION
```

这是必要修改，作用包括：

- `DIFY_BASE_PATH=` 时仍兼容官方根路径部署；
- 拒绝 `nuaai01` 这种缺少开头 `/` 的值；
- 拒绝 `/nuaai01/` 这种末尾带 `/` 的值；
- 为精确请求 `/nuaai01` 生成 Web location。

正确值：

```env
DIFY_BASE_PATH=/nuaai01
```

---

## 8. 增加环境变量示例

文件：

```text
docker/envs/core-services/web.env.example
```

增加：

```env
NEXT_PUBLIC_BASE_PATH=
```

文件：

```text
docker/envs/infrastructure/nginx.env.example
```

增加：

```env
DIFY_BASE_PATH=
```

这些修改用于记录服务级环境变量。实际部署值写在被 Git 忽略的 `docker/.env` 中。

---

## 9. 修改 Compose 模板

只修改：

```text
docker/docker-compose-template.yaml
```

不要直接把 `docker/docker-compose.yaml` 当作源文件修改。模板改完以后使用生成器生成本体。

### 9.1 Web 镜像参数化

找到：

```yaml
web:
  image: langgenius/dify-web:1.16.1
```

改为：

```yaml
web:
  image: ${DIFY_WEB_IMAGE:-langgenius/dify-web:1.16.1}
```

这是必要的部署修改。这样默认根路径部署仍可使用官方镜像，子路径部署可在 `.env` 中选择自定义镜像：

```env
DIFY_WEB_IMAGE=localhost/dify-web:1.16.1-nuaai01-aligned
```

不要把 `localhost/dify-web:...` 硬编码进模板。硬编码会导致其他机器只要没有这个本地镜像就无法部署。

### 9.2 Web 环境变量

在 `web.environment` 中增加：

```yaml
NEXT_PUBLIC_BASE_PATH: ${NEXT_PUBLIC_BASE_PATH:-}
```

该变量主要用于让运行环境与构建值一致。真正决定 Next.js basePath 的仍然是构建阶段。

### 9.3 内置 Nginx 环境变量

在 `nginx.environment` 中增加：

```yaml
DIFY_BASE_PATH: ${DIFY_BASE_PATH:-}
```

### 9.4 固定 PostgreSQL 部署时删除非目标数据库依赖

当前生产方案固定使用：

```env
DB_TYPE=postgresql
```

从以下服务的 `depends_on` 中删除非目标数据库项：

```text
api
api_websocket
worker
worker_beat
plugin_daemon
```

删除项包括：

```yaml
db_mysql:
  condition: service_healthy
  required: false

oceanbase:
  condition: service_healthy
  required: false

seekdb:
  condition: service_healthy
  required: false
```

`api_websocket` 官方只包含 `db_mysql`，只删除其 `db_mysql` 项。

这不是 `/nuaai01` 子路径所必需的修改，而是当前固定 PostgreSQL 拓扑与 `podman-compose 1.5.0` 的兼容修改。该版本在处理未启用 Profile、但被 `depends_on(required: false)` 引用的服务时可能出现：

```text
KeyError: 'db_mysql'
```

副作用：删除这些依赖后，如果以后将 `DB_TYPE` 改为 MySQL、OceanBase 或 SeekDB，必须先恢复相应服务依赖。否则启动顺序和健康检查关系不完整。

### 9.5 必须保留的 Compose 内容

不要删除以下 Profiles：

```yaml
api_websocket:
  profiles:
    - collaboration

db_postgres:
  profiles:
    - postgresql

weaviate:
  profiles:
    - weaviate
```

通过 `.env` 启用：

```env
COMPOSE_PROFILES=weaviate,postgresql,collaboration
```

不要为了绕过某一次启动问题永久删除 Profile。删除 Profile 会让这些可选服务在所有部署中无条件启动，破坏官方部署结构。

不要删除 API、Worker、Worker Beat 对 `init_permissions` 的官方依赖。删除后可能在存储目录权限尚未完成时启动应用。

不要展平 Agent Backend 的嵌套变量，例如不要把：

```yaml
${DIFY_AGENT_INNER_API_URL:-${PLUGIN_DIFY_INNER_API_URL:-http://api:5001}}
```

永久改为：

```yaml
${DIFY_AGENT_INNER_API_URL:-http://api:5001}
```

展平虽然可能绕过部分旧 Compose 解析器的问题，但会丢失对旧变量 `PLUGIN_DIFY_INNER_API_URL` 的兼容。

### 9.6 旧版 podman-compose 的 default 网络问题

`podman-compose 1.5.0` 在某些环境解析 Dify Compose 时可能报告：

```text
RuntimeError: missing networks: default
```

这是 Podman Compose 兼容问题，不是子路径修改。只有目标环境确实出现该错误时，才在模板底部增加：

```yaml
networks:
  default:
    driver: bridge
  ssrf_proxy_network:
    driver: bridge
    internal: true
```

注意：不要重复创建第二个 `networks:` 顶层键，而是在官方已有的 `networks:` 下增加 `default`。

显式 default 网络通常不会破坏 Docker Compose，但它仍属于特定 Compose 实现兼容修改。若部署环境不报错，可以不加，以减少与官方模板的差异。

### 9.7 init_permissions 与旧 Podman

旧版 `podman-compose/libpod` 可能把已经成功退出的 `init_permissions` 一次性容器视为“不在运行的依赖”，错误类似：

```text
container state improper
```

优先方案是使用能正确处理 `condition: service_completed_successfully` 的 Compose 实现，不修改官方依赖。

只有环境无法升级时，才考虑删除 API、Worker、Worker Beat 对 `init_permissions` 的依赖，并在启动应用前单独运行初始化服务。这个兼容方案的副作用是启动顺序由部署脚本负责，不能再依赖 Compose 自动保证。

---

## 10. 重新生成 docker-compose.yaml

执行：

```bash
cd /home/ubuntu/dify/docker
python3 generate_docker_compose
```

生成器会：

- 从 `docker-compose-template.yaml` 生成 `docker-compose.yaml`；
- 首次运行时从 `.env.example` 创建 `.env`；
- 首次运行时从各个 `*.env.example` 创建对应服务级 `.env` 文件；
- 已存在的本地环境文件不会被覆盖。

检查模板和生成文件同步：

```bash
cd /home/ubuntu/dify
git diff --check
```

---

## 11. 安装 Podman 部署环境

Ubuntu 示例：

```bash
sudo apt update
sudo apt install -y git podman podman-compose openssl
sudo loginctl enable-linger ubuntu
```

配置短镜像名称从 Docker Hub 解析：

```bash
sudo mkdir -p /etc/containers/registries.conf.d
printf '%s\n' 'unqualified-search-registries = ["docker.io"]' \
  | sudo tee /etc/containers/registries.conf.d/99-docker-io-search.conf
```

如果不配置，`redis:6-alpine`、`nginx:latest` 等镜像可能报：

```text
short-name ... did not resolve to an alias
```

这是 Podman 镜像解析配置，不需要修改 Dify 源码中的所有镜像名称。

---

## 12. 填写 docker/.env

假设生产入口：

```text
https://dify.example.com/nuaai01
```

编辑：

```bash
nano /home/ubuntu/dify/docker/.env
```

填写：

```env
CONSOLE_API_URL=https://dify.example.com/nuaai01
SERVER_CONSOLE_API_URL=http://api:5001
CONSOLE_WEB_URL=https://dify.example.com/nuaai01

APP_API_URL=https://dify.example.com/nuaai01
APP_WEB_URL=https://dify.example.com/nuaai01
SERVICE_API_URL=https://dify.example.com/nuaai01

FILES_URL=https://dify.example.com/nuaai01
INTERNAL_FILES_URL=http://api:5001

NEXT_PUBLIC_SOCKET_URL=wss://dify.example.com
NEXT_PUBLIC_BASE_PATH=/nuaai01
DIFY_BASE_PATH=/nuaai01

ENDPOINT_URL_TEMPLATE=https://dify.example.com/nuaai01/e/{hook_id}
TRIGGER_URL=https://dify.example.com/nuaai01

DIFY_WEB_IMAGE=localhost/dify-web:1.16.1-nuaai01-aligned

DB_TYPE=postgresql
VECTOR_STORE=weaviate
COMPOSE_PROFILES=weaviate,postgresql,collaboration

DIFY_AGENT_INNER_API_URL=http://api:5001
GRAPH_ENGINE_SCALE_UP_THRESHOLD=1

EXPOSE_NGINX_PORT=8080
NGINX_PORT=80
EXPOSE_NGINX_SSL_PORT=8444
```

Rootless Podman 不能默认绑定 443 这类低端口，所以即使内置 Nginx HTTPS 未启用，也将 Compose 的 SSL 暴露端口设为非特权端口 `8444`。

所有外部 URL 末尾不要加 `/`。

---

## 13. 设置统一且固定的 SECRET_KEY

进入 Docker 目录：

```bash
cd /home/ubuntu/dify/docker
```

生成一次密钥：

```bash
SECRET_VALUE=$(openssl rand -base64 42 | tr -d '\n')
```

把同一个值分别写入：

```text
docker/.env
docker/envs/security.env
```

命令：

```bash
sed -i "s|^SECRET_KEY=.*|SECRET_KEY=${SECRET_VALUE}|" .env
sed -i "s|^SECRET_KEY=.*|SECRET_KEY=${SECRET_VALUE}|" envs/security.env
unset SECRET_VALUE
```

这是正常部署步骤，不是可选项。`api` 负责签发 Token，`api_websocket` 负责校验 Token，两者必须使用同一个非空值。

如果浏览器出现：

```text
WebSocket connection error: Error: Connection rejected by server
```

同时 Nginx 日志已经返回 WebSocket 101，而 `docker_api_websocket_1` 日志出现：

```text
Socket authentication failed
Invalid token signature
```

说明代理和 Upgrade 已成功，真正的问题是 Token 签名。修复密钥后必须重新创建容器，并清除浏览器旧 Cookie/Local Storage 后重新登录。旧 Token 不会自动变成新签名。

不要在每次普通重启时重新随机生成 `SECRET_KEY`。频繁更换会使所有登录状态立即失效。

---

## 14. 构建自定义前端镜像

必须在仓库根目录执行：

```bash
cd /home/ubuntu/dify

podman build \
  -f web/Dockerfile \
  --build-arg NEXT_PUBLIC_BASE_PATH=/nuaai01 \
  -t localhost/dify-web:1.16.1-nuaai01-aligned \
  .
```

最后的 `.` 必须是 `/home/ubuntu/dify`。不能在 `/home/ubuntu/dify/web` 中执行相同命令，因为 Dockerfile 需要仓库根目录下的：

```text
pnpm-lock.yaml
pnpm-workspace.yaml
packages/
web/
```

错误构建上下文会出现：

```text
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml /app/
stat: /pnpm-lock.yaml: no such file or directory
```

如果构建过程收到：

```text
exit code 129
signal=SIGHUP
```

通常表示 SSH/终端挂断，而不是已经确认的源码编译错误。应在稳定终端、`tmux` 或构建系统中重新执行；Podman 会复用已完成缓存。

---

## 15. 导出、传输和导入镜像

构建服务器导出：

```bash
mkdir -p /home/ubuntu/dify/docker/images

podman save \
  --format oci-archive \
  -o /home/ubuntu/dify/docker/images/dify-web-1.16.1-nuaai01-aligned.oci.tar \
  localhost/dify-web:1.16.1-nuaai01-aligned
```

发送到部署服务器：

```bash
scp \
  /home/ubuntu/dify/docker/images/dify-web-1.16.1-nuaai01-aligned.oci.tar \
  ubuntu@DIFY服务器IP:/home/ubuntu/dify/docker/images/
```

部署服务器导入：

```bash
podman load \
  -i /home/ubuntu/dify/docker/images/dify-web-1.16.1-nuaai01-aligned.oci.tar
```

`docker/images/*.tar` 不应提交到 Git。镜像包体积大，应通过制品库、对象存储或 SCP 分发。

---

## 16. 启动 Podman Compose

先确认 `.env` 中包含：

```env
COMPOSE_PROFILES=weaviate,postgresql,collaboration
DIFY_WEB_IMAGE=localhost/dify-web:1.16.1-nuaai01-aligned
```

启动：

```bash
cd /home/ubuntu/dify/docker
podman-compose up -d
```

不要在同一套已经创建的容器上改用不同的 `--profile` 参数反复局部执行 `up`，旧版 podman-compose 可能尝试重复创建依赖容器。Profile 统一写在 `.env` 的 `COMPOSE_PROFILES` 中，从零一次启动整套服务。

---

## 17. 配置生产外层 Nginx

这一层是生产入口服务器上的原生 Nginx，不是 Dify Compose 内部的 Nginx，也不是测试服务器 8443 模拟配置。

只在生产域名已经存在的 `server {}` 中加入 `/nuaai01` location。域名、证书、监听端口和生产统一安全策略保持原配置。

```nginx
location = /nuaai01 {
    return 301 /nuaai01/;
}

location ^~ /nuaai01/ {
    proxy_pass http://DIFY服务器内网IP:8080;

    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    client_max_body_size 100m;
}
```

`proxy_pass` 后面不能加 `/`：

```nginx
proxy_pass http://DIFY服务器内网IP:8080;
```

外层必须保留 `/nuaai01`，由 Dify 内置 Nginx 再决定哪些路由删除前缀。

不要把测试环境的：

```text
listen 8443
proxy_pass http://127.0.0.1:8080
```

直接复制到生产。这只是单机模拟外层入口时使用的配置。

生产 Dify 服务器的 8080 应只允许入口 Nginx 服务器访问，不应向所有公网用户开放。

---

## 18. 最终验证

内部入口：

```bash
curl -i http://127.0.0.1:8080/nuaai01/console/api/setup
```

Socket.IO polling：

```bash
curl -i \
  'http://127.0.0.1:8080/nuaai01/socket.io/?EIO=4&transport=polling'
```

浏览器验证：

1. `https://生产域名/nuaai01/` 可以登录；
2. 页面跳转和刷新保留 `/nuaai01`；
3. 静态资源从 `/nuaai01/_next/...` 加载；
4. Console API 使用 `/nuaai01/console/api/...`；
5. App API 使用 `/nuaai01/api/...`；
6. 上传、下载和预览正常；
7. SSE 流式输出没有被缓冲；
8. Socket.IO 请求 `/nuaai01/socket.io/...`；
9. WebSocket Upgrade 返回 101；
10. 工作流协作不再出现 `Connection rejected by server`。

如果浏览器仍请求旧前端才存在的 API 或旧的 `/socket.io`，关闭旧页面，清除站点缓存或使用无痕窗口加载新镜像。

---

## 19. 修改分类与副作用汇总

| 修改 | 分类 | 是否必要 | 副作用或限制 |
|---|---|---:|---|
| 使用官方提交 `6f8ed69e` | 版本对齐 | 是 | 升级时需重新迁移 |
| 构建参数 `NEXT_PUBLIC_BASE_PATH=/nuaai01` | 子路径 | 是 | 镜像固定服务于该前缀 |
| WebSocket path 拼接 `basePath` | 子路径 | 是 | `NEXT_PUBLIC_SOCKET_URL` 只能填 Origin |
| Device Flow 拼接 `basePath` | 子路径 | 是 | 无根路径副作用，空 basePath 时仍为原路径 |
| 原生页面跳转拼接 `basePath` | 子路径 | 是 | 不能机械修改 Next Router 跳转 |
| 内置 Nginx API rewrite | 子路径 | 是 | API 删除前缀，Web 不能删除前缀 |
| `DIFY_BASE_PATH` 校验 | 子路径 | 是 | 配置不能以 `/` 结尾 |
| 删除 Docker DNS `127.0.0.11` | Podman 兼容 | Podman 必需 | Nginx 启动时需解析 api_websocket |
| Web 镜像参数化 | 部署 | 是 | `.env` 必须选择正确镜像 |
| 删除 MySQL/OceanBase/SeekDB 依赖 | 固定 PostgreSQL + 旧 Podman | 当前方案需要 | 切换数据库前必须恢复 |
| 显式声明 default 网络 | 旧 podman-compose 兼容 | 报错时才需要 | 增加与官方模板差异 |
| 删除 Profile | 临时规避手段 | 否 | 会让可选服务无条件启动 |
| 删除 init_permissions 依赖 | 临时规避手段 | 否 | 可能发生存储权限竞态 |
| 展平 Agent 嵌套变量 | 临时规避手段 | 否 | 丢失旧环境变量回退能力 |
| 固定统一 SECRET_KEY | 鉴权 | 是 | 更换后旧登录 Token 全部失效 |
| 修改 Dify 内层转发协议头 | 依赖生产拓扑 | 不在本方案修改 | 未知生产代理策略下可能产生错误或信任风险 |

---

## 20. 最终应修改的仓库文件

必要前端文件：

```text
web/app/components/workflow/collaboration/core/websocket-manager.ts
web/service/device-flow.ts
web/app/device/components/chooser.tsx
web/service/fetch.ts
web/app/components/goto-anything/actions/commands/account.tsx
web/app/components/goto-anything/actions/commands/go.tsx
```

必要部署文件：

```text
docker/nginx/conf.d/default.conf.template
docker/nginx/docker-entrypoint.sh
docker/envs/core-services/web.env.example
docker/envs/infrastructure/nginx.env.example
docker/docker-compose-template.yaml
docker/docker-compose.yaml
```

本地部署文件，不得提交密钥：

```text
docker/.env
docker/envs/security.env
docker/envs/**/*.env
docker/images/*.tar
```

完成所有修改后：

```bash
cd /home/ubuntu/dify/docker
python3 generate_docker_compose

cd /home/ubuntu/dify
git diff --check
git status --short
```
