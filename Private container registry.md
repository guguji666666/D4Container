🐳 私有 Docker Registry 部署指南（Cloudflare Tunnel + Token 模式，无 UI）

使用 Cloudflare Tunnel + Zero Trust 提供的 Token 接入，避免暴露公网端口，支持认证 + HTTPS 加密访问，无需本地证书挂载。

⸻

📁 项目结构一览

/root/data/docker_data/
├── docker-registry/
│   ├── docker-compose.yaml     # Registry 服务定义
│   ├── auth/                   # htpasswd 密码文件
│   └── data/                   # 镜像数据目录
├── cloudflared/
│   ├── docker-compose.yaml     # cloudflared Tunnel 服务


⸻

🧱 步骤一：创建目录结构

mkdir -p /root/data/docker_data/docker-registry/auth
mkdir -p /root/data/docker_data/docker-registry/data
mkdir -p /root/data/docker_data/cloudflared


⸻

🔐 步骤二：创建认证文件（htpasswd）

cd /root/data/docker_data/docker-registry

USERNAME=guguji
PASSWORD=yourpassword

docker run --rm \
  --entrypoint htpasswd \
  httpd:2 \
  -Bbn "$USERNAME" "$PASSWORD" > auth/htpasswd


⸻

🛠️ 步骤三：配置 Registry 服务（docker-registry/docker-compose.yaml）

version: '3.8'

services:
  registry:
    image: registry:3
    container_name: docker-registry
    restart: always
    ports:
      - "127.0.0.1:5000:5000"  # ⚠️ 仅本地监听，供 tunnel 转发
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth


⸻

🚇 步骤四：配置 Cloudflare Tunnel（cloudflared/docker-compose.yaml）

version: '3.8'

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel --no-autoupdate run --token <YOUR-CLOUDFLARE-TUNNEL-TOKEN>

💡 你可以从 Cloudflare Zero Trust 仪表盘添加 Tunnel，并获取专属 Token。
例如：

https://dash.cloudflare.com > Zero Trust > Access > Tunnels > Create Tunnel



⸻

🚀 启动服务

cd /root/data/docker_data/docker-registry
docker compose up -d

cd /root/data/docker_data/cloudflared
docker compose up -d


⸻

🔐 登录私有仓库

docker login https://registry.example.com


⸻

📦 推送和拉取镜像

docker tag nginx registry.example.com/nginx-gugu
docker push registry.example.com/nginx-gugu
docker pull registry.example.com/nginx-gugu


⸻

🔎 查看镜像列表

curl -u guguji:yourpassword https://registry.example.com/v2/_catalog


⸻

✅ 总结

特性	状态
TLS 加密	✅ Cloudflare 自动处理
本地监听端口	✅ 避免公网暴露
无需证书文件或凭据挂载	✅ 使用 --token 模式
支持身份验证（htpasswd）	✅
支持 push/pull	✅
支持 curl API 查询	✅


⸻

需要我将这份教程导出为 .md 文件吗？或者要不要我附一段创建 Cloudflare Tunnel 的 GUI 操作说明？
