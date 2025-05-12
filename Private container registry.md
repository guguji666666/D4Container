# 🐳 私有 Docker Registry 部署文档（Cloudflare Tunnel + Token 模式）

使用 Cloudflare Tunnel + Zero Trust Token 模式将私有 Docker 镜像仓库绑定到公网域名，实现 HTTPS 加密访问和认证保护，无需暴露真实 IP，无需本地挂载证书。

---

## 📁 项目目录结构

```
/root/data/docker_data/
├── docker-registry/
│   ├── docker-compose.yaml     # Registry 服务定义
│   ├── auth/                   # htpasswd 密码文件
│   └── data/                   # 镜像数据目录
└── cloudflared/
    └── docker-compose.yaml     # Cloudflare Tunnel 服务
```

---

## 🧱 步骤一：创建目录

```bash
mkdir -p /root/data/docker_data/docker-registry/auth
mkdir -p /root/data/docker_data/docker-registry/data
mkdir -p /root/data/docker_data/cloudflared
```

---

## 🔐 步骤二：创建登录认证文件（htpasswd）

```bash
cd /root/data/docker_data/docker-registry

USERNAME=guguji
PASSWORD=yourpassword

docker run --rm \
  --entrypoint htpasswd \
  httpd:2 \
  -Bbn "$USERNAME" "$PASSWORD" > auth/htpasswd
```

> 📌 可多次运行该命令添加多个用户（推荐使用 `>>` 而非 `>` 覆盖）

---

## 🛠️ 步骤三：配置 Registry 服务

路径：`/root/data/docker_data/docker-registry/docker-compose.yaml`

```yaml
version: '3.8'

services:
  registry:
    image: registry:3
    container_name: docker-registry
    restart: always
    ports:
      - "127.0.0.1:5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth
```

---

## 🚇 步骤四：配置 Cloudflare Tunnel 服务

路径：`/root/data/docker_data/cloudflared/docker-compose.yaml`

```yaml
version: '3.8'

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel --no-autoupdate run --token <YOUR-CLOUDFLARE-TUNNEL-TOKEN>
```

> 🔑 替换 `<YOUR-CLOUDFLARE-TUNNEL-TOKEN>` 为你在 Cloudflare Zero Trust 仪表盘获取的 token。

你可以在此处生成：
https://dash.cloudflare.com > Zero Trust > Access > Tunnels > Create Tunnel

---

## 🚀 启动服务

```bash
cd /root/data/docker_data/docker-registry
docker compose up -d

cd /root/data/docker_data/cloudflared
docker compose up -d
```

---

## 🔐 登录你的私有仓库

```bash
docker login https://registry.example.com
```

---

## 📦 镜像推送与拉取示例

```bash
docker tag nginx registry.example.com/nginx-gugu
docker push registry.example.com/nginx-gugu
docker pull registry.example.com/nginx-gugu
```

---

## 🔎 查看已有镜像

```bash
curl -u guguji:yourpassword https://registry.example.com/v2/_catalog
```

---

## 🧹 可选：清理无用资源

```bash
docker system prune -af
docker volume prune -f
docker network prune -f
```

---

## ✅ 部署要点总结

| 项目                           | 状态                  |
| ------------------------------ | --------------------- |
| TLS 加密                       | ✅ Cloudflare 自动处理 |
| 本地监听避免公网暴露           | ✅                     |
| 用户认证（htpasswd）           | ✅                     |
| 支持 curl 查询镜像列表         | ✅                     |
| 支持 push/pull 操作            | ✅                     |
| 无需本地证书或 tunnel 凭据挂载 | ✅ 使用 token 模式     |
