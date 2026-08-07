# Dify 1.16.1 `/nuaai01` 快速部署指南

## 1. 安装软件并克隆仓库

```bash
sudo apt update
sudo apt install -y git podman podman-compose openssl
sudo loginctl enable-linger ubuntu

git clone \
  --branch release-1.16.1-nuaai01 \
  --single-branch \
  git@github.com:kiang115/kiang-dify.git \
  /home/ubuntu/dify

cd /home/ubuntu/dify
```

配置 Podman 镜像源：

```bash
sudo mkdir -p /etc/containers/registries.conf.d
printf '%s\n' 'unqualified-search-registries = ["docker.io"]' \
  | sudo tee /etc/containers/registries.conf.d/99-docker-io-search.conf
```

## 2. 生成 Compose 和环境文件

```bash
cd /home/ubuntu/dify/docker
python3 generate_docker_compose
```

## 3. 填写部署参数

把 `PUBLIC_ORIGIN` 改成实际生产入口，末尾不要加 `/`：

```bash
PUBLIC_ORIGIN='https://你的生产域名'
BASE_PATH='/nuaai01'
WEB_IMAGE='localhost/dify-web:1.16.1-nuaai01-aligned'

if [[ "$PUBLIC_ORIGIN" == https://* ]]; then
  SOCKET_ORIGIN="wss://${PUBLIC_ORIGIN#https://}"
else
  SOCKET_ORIGIN="ws://${PUBLIC_ORIGIN#http://}"
fi

set_env() {
  local file="$1"
  local key="$2"
  local value="$3"
  if grep -q "^${key}=" "$file"; then
    sed -i "s|^${key}=.*|${key}=${value}|" "$file"
  else
    printf '%s=%s\n' "$key" "$value" >> "$file"
  fi
}

cd /home/ubuntu/dify/docker

set_env .env CONSOLE_API_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env SERVER_CONSOLE_API_URL 'http://api:5001'
set_env .env CONSOLE_WEB_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env APP_API_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env APP_WEB_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env SERVICE_API_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env FILES_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env INTERNAL_FILES_URL 'http://api:5001'
set_env .env NEXT_PUBLIC_SOCKET_URL "$SOCKET_ORIGIN"
set_env .env NEXT_PUBLIC_BASE_PATH "$BASE_PATH"
set_env .env DIFY_BASE_PATH "$BASE_PATH"
set_env .env ENDPOINT_URL_TEMPLATE "${PUBLIC_ORIGIN}${BASE_PATH}/e/{hook_id}"
set_env .env TRIGGER_URL "${PUBLIC_ORIGIN}${BASE_PATH}"
set_env .env DIFY_WEB_IMAGE "$WEB_IMAGE"
set_env .env DB_TYPE 'postgresql'
set_env .env VECTOR_STORE 'weaviate'
set_env .env COMPOSE_PROFILES 'weaviate,postgresql,collaboration'
set_env .env DIFY_AGENT_INNER_API_URL 'http://api:5001'
set_env .env GRAPH_ENGINE_SCALE_UP_THRESHOLD '1'
set_env .env EXPOSE_NGINX_PORT '8080'
set_env .env NGINX_PORT '80'
set_env .env EXPOSE_NGINX_SSL_PORT '8444'
```

生成固定 `SECRET_KEY`，并把同一个值写入 API 和 WebSocket 使用的两个文件：

```bash
SECRET_VALUE=$(openssl rand -base64 42 | tr -d '\n')
set_env .env SECRET_KEY "$SECRET_VALUE"
set_env envs/security.env SECRET_KEY "$SECRET_VALUE"
unset SECRET_VALUE SOCKET_ORIGIN
```

## 4. 构建前端镜像

必须在仓库根目录执行：

```bash
cd /home/ubuntu/dify

podman build \
  -f web/Dockerfile \
  --build-arg NEXT_PUBLIC_BASE_PATH=/nuaai01 \
  -t localhost/dify-web:1.16.1-nuaai01-aligned \
  .
```

如果在另一台服务器构建，构建完成后导出：

```bash
podman save \
  --format oci-archive \
  -o /tmp/dify-web-1.16.1-nuaai01-aligned.oci.tar \
  localhost/dify-web:1.16.1-nuaai01-aligned
```

传到 Dify 服务器：

```bash
scp /tmp/dify-web-1.16.1-nuaai01-aligned.oci.tar \
  ubuntu@DIFY服务器IP:/tmp/
```

在 Dify 服务器导入：

```bash
podman load -i /tmp/dify-web-1.16.1-nuaai01-aligned.oci.tar
```

## 5. 启动 Dify

```bash
cd /home/ubuntu/dify/docker
podman-compose up -d
```

## 6. 配置入口服务器 Nginx

在生产域名现有的 `server {}` 中加入以下内容，把 `DIFY服务器内网IP` 替换为实际地址：

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

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 7. 更新代码后重新部署

```bash
cd /home/ubuntu/dify
git switch release-1.16.1-nuaai01
git pull --ff-only origin release-1.16.1-nuaai01

cd /home/ubuntu/dify/docker
python3 generate_docker_compose
podman-compose down
podman-compose up -d
```

前端源码变化后，先重新执行第 4 步构建或导入镜像，再重新创建容器。
