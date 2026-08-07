# Dify `/nuaai01` 子路径部署改造记录

本文记录基于 Dify `main` 分支、原始提交 `12300f3833`（Dify 1.16.1）完成的全部仓库修改、部署背景、操作历史、当前状态以及在生产服务器上的复现步骤。

## 1. 最终目标与访问链路

生产访问链路：

```text
外部用户
  ↓
https://生产域名/nuaai01
  ↓
入口服务器上的原生 Nginx
  ↓
http://192.168.110.186:8080
  ↓
Podman/Podman Compose 部署的 Dify 内置 Nginx
  ├─ Web → web:3000
  ├─ API → api:5001
  ├─ WebSocket → api_websocket:5001
  └─ 插件端点 → plugin_daemon:5002
```

目标要求：

- 主界面、页面跳转和刷新始终保留 `/nuaai01`。
- Next.js 静态资源、字体、图片和 Monaco 等资源正常加载。
- Console API、App API、Service API、OpenAPI、文件接口均通过 `/nuaai01` 访问。
- 文件上传、下载、预览、SSE 和 Socket.IO/WebSocket 正常。
- 8080 内网入口不能由外部用户直接访问。
- 最终部署可以迁移到仅支持 Podman 的生产服务器。

## 2. 为什么必须重新构建前端

`web/Dockerfile` 中的 `NEXT_PUBLIC_BASE_PATH` 是构建参数，并在 `pnpm build` 前写入环境：

```dockerfile
ARG NEXT_PUBLIC_BASE_PATH=""
ENV NEXT_PUBLIC_BASE_PATH="$NEXT_PUBLIC_BASE_PATH"
```

`web/next.config.ts` 将它交给 Next.js：

```ts
basePath: env.NEXT_PUBLIC_BASE_PATH
```

因此官方预构建镜像 `langgenius/dify-web:1.16.1` 的页面路由和 `/_next` 静态资源已经按根路径生成。仅在容器启动时设置 `NEXT_PUBLIC_BASE_PATH=/nuaai01` 不能完整改变这些构建产物，必须从仓库根目录重新构建 Web 镜像。

## 3. 仓库文件修改明细

### 3.1 WebSocket 路径适配

文件：`web/app/components/workflow/collaboration/core/websocket-manager.ts`

新增：

```ts
import { basePath } from '@/utils/var'
```

删除：

```ts
path: '/socket.io'
```

改为：

```ts
path: `${basePath}/socket.io`
```

原因：Socket.IO 客户端的 `path` 不会自动继承 Next.js 的 `basePath`。生产环境必须请求 `/nuaai01/socket.io`；根路径部署时 `basePath` 为空，仍请求 `/socket.io`。

### 3.2 Device Flow API 请求适配

文件：`web/service/device-flow.ts`

新增：

```ts
import { basePath } from '@/utils/var'
```

删除：

```ts
const DEVICE_BASE = '/openapi/v1/oauth/device'
```

改为：

```ts
const DEVICE_BASE = `${basePath}/openapi/v1/oauth/device`
```

原因：该模块直接调用浏览器 `fetch`，不会自动附加 Next.js 的 `basePath`。修改后 lookup、approve、deny、approval-context 和 approve-external 均通过 `/nuaai01/openapi/...` 访问。

### 3.3 Device Flow SSO 浏览器跳转适配

文件：`web/app/device/components/chooser.tsx`

新增：

```ts
import { basePath } from '@/utils/var'
```

删除：

```ts
window.location.href = `/openapi/v1/oauth/device/sso-initiate?...`
```

改为：

```ts
window.location.href = `${basePath}/openapi/v1/oauth/device/sso-initiate?...`
```

原因：`window.location.href` 是原生浏览器跳转，不受 Next.js 路由器控制；不加前缀会跳到生产域名根目录的 `/openapi`。

### 3.4 已初始化系统的登录跳转适配

文件：`web/service/fetch.ts`

新增：

```ts
import { basePath } from '@/utils/var'
```

删除：

```ts
globalThis.location.href = `${globalThis.location.origin}/signin`
```

改为：

```ts
globalThis.location.href = `${globalThis.location.origin}${basePath}/signin`
```

原因：当接口返回 `403 already_setup` 时，原代码会跳转到根目录 `/signin`，从而丢失 `/nuaai01`。

### 3.5 Goto Anything 账户页跳转适配

文件：`web/app/components/goto-anything/actions/commands/account.tsx`

新增：

```ts
import { basePath } from '@/utils/var'
```

文件内两处删除：

```ts
window.location.href = '/account'
```

改为：

```ts
window.location.href = `${basePath}/account`
```

原因：直接执行 Account 命令和注册后的 `navigation.account` 命令都使用原生浏览器跳转，必须显式附加子路径。

### 3.6 Goto Anything 普通页面跳转适配

文件：`web/app/components/goto-anything/actions/commands/go.tsx`

新增：

```ts
import { basePath } from '@/utils/var'
```

删除：

```ts
if (args?.path) window.location.href = args.path
```

改为：

```ts
if (args?.path) window.location.href = `${basePath}${args.path}`
```

原因：该命令中的 Apps、Knowledge、Plugins、Tools、Explore 和 Account 路径均以 `/` 开头，并通过 `window.location.href` 跳转。修改后这些普通页面不会离开 `/nuaai01`。

说明：Next.js 的 `<Link href="/...">` 和项目封装的 `router.push('/...')` 会由 Next.js 自动处理 `basePath`，不能机械地重复添加，否则可能形成双前缀；本次只修改绕过 Next.js 路由器的确定位置。

### 3.7 Dify 内置 Nginx 子路径路由

文件：`docker/nginx/conf.d/default.conf.template`

所有入口路径前增加 `${DIFY_BASE_PATH}`：

```text
/console/api → ${DIFY_BASE_PATH}/console/api
/api         → ${DIFY_BASE_PATH}/api
/socket.io   → ${DIFY_BASE_PATH}/socket.io
/v1          → ${DIFY_BASE_PATH}/v1
/openapi     → ${DIFY_BASE_PATH}/openapi
/files       → ${DIFY_BASE_PATH}/files
/explore     → ${DIFY_BASE_PATH}/explore
/e           → ${DIFY_BASE_PATH}/e
/mcp         → ${DIFY_BASE_PATH}/mcp
/triggers    → ${DIFY_BASE_PATH}/triggers
/            → ${DIFY_BASE_PATH}/
```

API、Socket.IO、插件、MCP 和触发器路由新增：

```nginx
rewrite ^${DIFY_BASE_PATH}(/.*)$ $1 break;
```

原因：这些后端服务仍注册在根路径，需要在转发到对应容器前删除 `/nuaai01`。

Web 路由不执行 rewrite：

```nginx
location ${DIFY_BASE_PATH}/ {
  proxy_pass http://web:3000;
  include proxy.conf;
}
```

原因：以 `NEXT_PUBLIC_BASE_PATH=/nuaai01` 构建的 Next.js 服务必须看到原始 `/nuaai01/...` URI。

模板中还新增：

```nginx
${BASE_PATH_EXACT_LOCATION}
```

用于处理没有尾斜杠的精确 `/nuaai01` 请求。

当 `DIFY_BASE_PATH` 为空时，模板展开回原来的根路径路由，继续兼容默认部署。

### 3.8 Dify 内置 Nginx 环境变量校验

文件：`docker/nginx/docker-entrypoint.sh`

新增 `DIFY_BASE_PATH` 初始化、格式校验以及精确 location 生成逻辑：

```sh
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

允许值：空字符串或类似 `/nuaai01` 的路径。禁止缺少开头 `/` 或以 `/` 结尾，避免生成歧义或无效 Nginx 配置。

### 3.9 Compose Web 和 Nginx 环境变量

文件：

- `docker/docker-compose-template.yaml`
- `docker/docker-compose.yaml`

Web 服务新增：

```yaml
NEXT_PUBLIC_BASE_PATH: ${NEXT_PUBLIC_BASE_PATH:-}
```

Nginx 服务新增：

```yaml
DIFY_BASE_PATH: ${DIFY_BASE_PATH:-}
```

原因：分别把子路径传给 Web 运行环境和 Dify 内置 Nginx 模板。

两个 Compose 文件底部还新增：

```yaml
networks:
  default:
    driver: bridge
```

原因：`podman-compose 1.5.0` 解析原文件时报告 `RuntimeError: missing networks: default`。Docker Compose 会隐式创建 default 网络，但当前 Podman Compose 要求在多个服务显式引用它时也显式声明。

### 3.10 环境变量示例文件

文件：`docker/envs/core-services/web.env.example`

新增：

```env
NEXT_PUBLIC_BASE_PATH=
```

文件：`docker/envs/infrastructure/nginx.env.example`

新增：

```env
DIFY_BASE_PATH=
```

原因：记录新变量并支持服务级 env 文件部署。按照仓库约定，没有把可选子路径变量加入 `docker/.env.example`。

## 4. 本地未提交环境配置

`docker/.env` 是本地忽略文件，不会进入 Git 提交。测试环境当前已设置：

```env
NEXT_PUBLIC_BASE_PATH=/nuaai01
DIFY_BASE_PATH=/nuaai01
EXPOSE_NGINX_PORT=8080
NGINX_PORT=80
```

生产部署建议补充以下变量，所有外部 URL 末尾不要加 `/`：

```env
NEXT_PUBLIC_BASE_PATH=/nuaai01
DIFY_BASE_PATH=/nuaai01

CONSOLE_API_URL=https://生产域名/nuaai01
SERVER_CONSOLE_API_URL=http://api:5001

CONSOLE_WEB_URL=https://生产域名/nuaai01
APP_API_URL=https://生产域名/nuaai01
APP_WEB_URL=https://生产域名/nuaai01
SERVICE_API_URL=https://生产域名/nuaai01

FILES_URL=https://生产域名/nuaai01
INTERNAL_FILES_URL=http://api:5001

NEXT_PUBLIC_SOCKET_URL=wss://生产域名

ENDPOINT_URL_TEMPLATE=https://生产域名/nuaai01/e/{hook_id}
TRIGGER_URL=https://生产域名/nuaai01

EXPOSE_NGINX_PORT=8080
NGINX_PORT=80
```

`NEXT_PUBLIC_SOCKET_URL` 只填写 Origin。Socket.IO 的 `/nuaai01/socket.io` 由前端代码中的 `path` 单独提供，不能把 path 再拼进该变量。

## 5. 前端镜像构建与 Compose 切换

必须以仓库根目录 `/home/ubuntu/dify` 为构建上下文：

```bash
cd /home/ubuntu/dify

podman build \
  -f web/Dockerfile \
  --build-arg NEXT_PUBLIC_BASE_PATH=/nuaai01 \
  -t localhost/dify-web:1.16.1-nuaai01 \
  .
```

不能在 `/home/ubuntu/dify/web` 下使用 `.` 作为上下文，否则 Dockerfile 无法复制根目录的 `pnpm-lock.yaml`、workspace 文件、`packages/` 和 `e2e/`。

如果 SSH 可能断开，可保存日志后台构建：

```bash
cd /home/ubuntu/dify
nohup podman build \
  -f web/Dockerfile \
  --build-arg NEXT_PUBLIC_BASE_PATH=/nuaai01 \
  -t localhost/dify-web:1.16.1-nuaai01 \
  . > /tmp/dify-web-build.log 2>&1 &
```

查看日志：

```bash
tail -f /tmp/dify-web-build.log
```

构建完成后必须把 Compose 的 Web 镜像从官方镜像切换为：

```yaml
image: localhost/dify-web:1.16.1-nuaai01
```

当前仓库尚未完成这一步；两个 Compose 文件当前仍然是 `langgenius/dify-web:1.16.1`。

## 6. 外层 Nginx 配置方向

生产外层 Nginx 必须保留 `/nuaai01`，不要在第一层删除前缀：

```nginx
location = /nuaai01 {
    return 301 /nuaai01/;
}

location ^~ /nuaai01/ {
    proxy_pass http://192.168.110.186:8080;

    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    client_max_body_size 100m;
}
```

测试服务器上的外层 Nginx 使用 `8443` 模拟入口，当前是纯 HTTP，没有 TLS。

## 7. 已执行的系统操作

测试服务器初始环境：Ubuntu 26.04 LTS、3.6 GiB 内存、2 GiB Swap，约 40 GiB 可用磁盘。

已经安装：

```text
nginx 1.28.3
podman 5.7.0
podman-compose 1.5.0
```

Podman 当前特征：rootless、overlay 存储驱动、runc runtime。

系统原生 Nginx 默认监听端口已由 80 改为 8443：

```text
0.0.0.0:8443
[::]:8443
```

原配置备份：

```text
/etc/nginx/sites-available/default.before-8443
```

已确认当时 `http://127.0.0.1:8443/` 返回 Ubuntu 默认页面 HTTP 200，80 端口已释放。尚未把 `/nuaai01` 代理配置写入该系统 Nginx。

## 8. 操作历史与构建现状

1. 检查了 Dify Compose 路由结构和 Web Dockerfile，确认 `NEXT_PUBLIC_BASE_PATH` 必须在构建期设置。
2. 完成前端 WebSocket、Device Flow、登录跳转和 Goto Anything 普通页面跳转修改。
3. 完成 Dify 内置 Nginx 的子路径路由、去前缀以及变量校验修改。
4. 完成 Compose 变量透传和 Podman Compose default 网络显式声明。
5. 第一次前端构建曾进入 `Creating an optimized production build ...`，随后会话中断。
6. 有一次在 `/home/ubuntu/dify/web` 下构建，因错误的构建上下文缺少根目录 `pnpm-lock.yaml` 而失败。
7. 从仓库根目录再次构建时，`pnpm build` 收到 `SIGHUP`，退出码 129。该状态表示终端或父会话挂断，不是已确认的前端编译错误；若是常见 OOM，通常会看到 SIGKILL/137。
8. 尚未确认 `localhost/dify-web:1.16.1-nuaai01` 镜像成功生成。
9. 尚未启动 Dify 容器，也没有把 Compose Web 镜像切换到自定义镜像。

## 9. 尚未执行的测试和验证

按要求，当前阶段没有运行：

- Vitest；
- TypeScript 类型检查；
- lint；
- 后端测试；
- E2E 测试；
- Dify 启动验证。

现有 WebSocket 和 Device Flow 部分测试仍断言修改前的根路径，后续允许测试时需要同步更新测试预期。

## 10. 已知待办和风险

### 10.1 Compose 尚未使用自定义 Web 镜像

构建成功后必须切换 `web.image`，否则启动的仍是没有 Next.js `/nuaai01` 构建产物的官方镜像。

### 10.2 双层 Nginx 的转发协议头

生产请求先经过 HTTPS 外层 Nginx，再通过 HTTP 进入 Dify 内置 Nginx。当前 `docker/nginx/proxy.conf.template` 使用内层的 `$scheme` 和 `$server_port` 重写：

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Port $server_port;
```

因此外层传入的 `https/443` 可能在第二层被覆盖为 `http/80`。插件路由中的 `Dify-Hook-Url` 也使用内层 `$scheme`。该问题目前按决定暂不修改，但生产上线前应结合“8080 只允许可信入口服务器访问”处理可信转发头。

### 10.3 8080 的访问边界

Compose 的普通端口映射默认可能监听 `0.0.0.0:8080`。生产环境应通过宿主机绑定地址和/或防火墙，仅允许外层入口 Nginx 所在服务器访问 8080，避免外部用户绕过域名、TLS 和 `/nuaai01` 入口。

### 10.4 Podman Compose 配置解析尚未复查

加入显式 default 网络后，尚未再次执行 `podman-compose ... config`。后续允许验证时应确认问题已经解决。

## 11. 推荐的后续顺序

1. 在稳定的 `tmux`、`screen` 或 `nohup` 会话中完成前端镜像构建。
2. 确认自定义 Web 镜像存在。
3. 将 Compose Web 镜像切换为自定义镜像。
4. 填写测试服务器实际入口地址对应的 URL 环境变量。
5. 解析 Podman Compose 配置。
6. 启动 Dify 并先验证宿主机 8080。
7. 配置测试服务器原生 Nginx 的 `/nuaai01` 代理。
8. 验证页面、刷新、静态资源、Console API、App API、文件、SSE 和 WebSocket。
9. 上生产前处理可信转发协议头及 8080 防火墙白名单。

