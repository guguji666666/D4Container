# 🐳 私有 Docker Registry 部署文档（Cloudflare Tunnel + Token 模式，无 UI）

> 使用 Cloudflare Tunnel 绑定域名并启用 HTTPS 加密，不暴露真实公网 IP，无需本地证书挂载，支持认证与 push/pull。

---

## 📁 项目目录结构

```bash
/root/data/docker_data/
├── docker-registry/
│   ├── docker-compose.yaml     # Registry 服务定义
│   ├── auth/                   # htpasswd 密码文件
│   └── data/                   # 镜像数据目录
├── cloudflared/
│   └── docker-compose.yaml     # Cloudflare Tunnel 服务


⸻

🧱 步骤一：创建目录

mkdir -p /root/data/docker_data/docker-registry/auth
mkdir -p /root/data/docker_data/docker-registry/data
mkdir -p /root/data/docker_data/cloudflared


⸻

🔐 步骤二：创建登录认证文件（htpasswd）

cd /root/data/docker_data/docker-registry

USERNAME=guguji
PASSWORD=yourpassword

docker run --rm \
  --entrypoint htpasswd \
  httpd:2 \
  -Bbn "$USERNAME" "$PASSWORD" > auth/htpasswd

📌 可多次运行该命令添加多个用户（推荐使用 >> 而非 > 覆盖）

⸻

🛠️ 步骤三：配置 Registry 服务（docker-compose.yaml）

路径：/root/data/docker_data/docker-registry/docker-compose.yaml

version: '3.8'

services:
  registry:
    image: registry:3
    container_name: docker-registry
    restart: always
    ports:
      - "127.0.0.1:5000:5000"  # 本地监听，供 cloudflared 反代
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth


⸻

🚇 步骤四：配置 Cloudflare Tunnel 服务（Token 模式）

路径：/root/data/docker_data/cloudflared/docker-compose.yaml

version: '3.8'

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel --no-autoupdate run --token <YOUR-CLOUDFLARE-TUNNEL-TOKEN>

🔑 替换 <YOUR-CLOUDFLARE-TUNNEL-TOKEN> 为你在 Cloudflare Zero Trust 仪表盘获取的 token。

你可以在此处生成：
https://dash.cloudflare.com > Zero Trust > Access > Tunnels > Create Tunnel

⸻

🚀 启动服务

cd /root/data/docker_data/docker-registry
docker compose up -d

cd /root/data/docker_data/cloudflared
docker compose up -d


⸻

🔐 登录你的私有仓库

docker login https://registry.example.com

输入你设置的用户名与密码。

⸻

📦 镜像推送与拉取示例

# 打标签
docker tag nginx registry.example.com/nginx-gugu

# 推送镜像
docker push registry.example.com/nginx-gugu

# 拉取镜像
docker pull registry.example.com/nginx-gugu


⸻

🔎 查看已有镜像

curl -u guguji:yourpassword https://registry.example.com/v2/_catalog


⸻

🧹 可选：清理无用资源

docker system prune -af
docker volume prune -f
docker network prune -f


⸻

✅ 部署要点总结

项目	状态
TLS 加密	✅ Cloudflare 自动处理
本地监听避免公网暴露	✅
用户认证（htpasswd）	✅
支持 curl 查询镜像列表	✅
支持 push/pull 操作	✅
无需本地证书或 tunnel 凭据挂载	✅ 使用 token 模式
