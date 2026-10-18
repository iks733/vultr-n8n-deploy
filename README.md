# Vultr部署n8n：从选套餐到 HTTPS 上线的完整流程，含 $6 起的套餐价格与避坑清单

在 Vultr 上部署 n8n，最省事的路径是：开一台 Cloud Compute 云服务器，装好 Docker，用 Docker Compose 同时拉起 n8n 和 Traefik，HTTPS 证书由 Traefik 自动向 Let's Encrypt 申请，全程不用手动碰证书文件。入门配置 $6/月 就能跑起来，教程里更常推荐的是 2 vCPU / 2GB 内存的 $12/月 方案。下面把选套餐、部署、避坑三件事一次讲清楚。

> **选购速览**：个人自动化，$6/月 的入门套餐足够；要跑 AI agent 或大量并发工作流，直接上 $12/月 的 2 vCPU / 2GB 方案。部署用 Docker Compose + Traefik，证书全自动。域名记得提前把 A 记录指到服务器 IP。👉 [前往 Vultr 开通云服务器，照下文步骤部署 n8n](https://bit.ly/VulTR)

## 为什么把 n8n 放到 Vultr 上自己跑

先算钱。n8n Cloud 托管版每月 $20–50，一年就是 $240–600；同样的工作流放到 Vultr 上自托管，入门方案 $6/月、推荐配置 $12/月，一年 $72–144，差价两年下来够买一台新服务器。钱之外还有三个实际差别：数据完全在自己服务器上，工作流和凭证不出门；没有执行次数限制——n8n Cloud 在低价套餐上会限制执行量；想装什么社区节点自己说了算。

如果你现在的主力是 Zapier 或 Make，迁到自托管 n8n 通常就是为了躲开那几件事：任务数配额、处理大 JSON 数据时的负载限制、webhook 触发延迟，以及敏感数据和 API 密钥过第三方服务器的问题。n8n 本身是开源的节点式自动化引擎，提供 400 多个集成，接 HTTP、数据库、SaaS API 和 AI 服务都有现成节点。

自托管当然也有代价：维护、备份、证书这些事归你管。好在下面这些都能靠脚本和自动配置一次解决。

## 跑 n8n 该选哪一档 Vultr 套餐

主流选择集中在 Cloud Compute 这一档，配置够不够主要看内存。各档位对比如下：

| 方案 | 配置 | 月付价格 | 适合场景 | 购买入口 |
| --- | --- | --- | --- | --- |
| Cloud Compute（Regular Performance）入门 | 1 vCPU / 1GB 内存 | $6/月 | 个人工作流、少量 webhook | [ 开通入门套餐](https://bit.ly/VulTR) |
| Cloud Compute（Regular Performance）推荐 | 2 vCPU / 2GB 内存 | $12/月 | 常驻自动化、AI 节点 | [ 开通 2GB 推荐方案](https://bit.ly/VulTR) |
| High Frequency Compute | NVMe 存储 + 高主频 CPU | 按所选配置计价 | 处理大 JSON、对脚本速度敏感 | [ 查看高频型方案](https://bit.ly/VulTR) |
| Optimized Cloud Compute | 更强算力，配置自选 | 按所选配置计价 | 重度工作负载 | [ 查看高性能方案](https://bit.ly/VulTR) |

对应的决策规则：

- 只跑个人自动化、几十个 webhook 以内 → $6 入门档够用
- 要跑内存占用大的 JS 节点或 LangChain 这类 AI agent → 上 2GB 的 $12 方案
- 数据管道很重、对 CPU 主频敏感 → 选 High Frequency 或 Optimized 按需配

机器位置选离你的数据库和主要 API 近的机房，webhook 的往返延机会直接体现在整条自动化链路的响应速度上。新用户注册时落地页如果显示试用额度，可以先拿额度把环境跑通再决定长期方案。👉 [开通 $12/月 的 2 vCPU / 2GB 方案](https://bit.ly/VulTR)

## Vultr 部署 n8n 的完整步骤

整条流程是：开服务器 → 装 Docker → 配目录和环境变量 → 写 docker-compose.yaml → 启动 → 解析域名打开后台，全程大概十几分钟。以 Ubuntu 24.04 + Traefik 这条官方文档路线为主线。

1. 开服务器。选 Cloud Compute，系统选 Ubuntu 24.04 LTS，添加 SSH key，选好机房后部署。👉 [开一台预装 Ubuntu 24.04 的 Vultr 云服务器](https://bit.ly/VulTR)
2. SSH 进去装 Docker：

bash
apt update && apt upgrade -y
apt install -y docker.io docker-compose curl
systemctl enable docker && systemctl start docker


3. 准备目录和环境变量：

bash
mkdir -p ~/n8n-automation/n8n-data
cd ~/n8n-automation
sudo chown -R 1000:1000 n8n-data
nano .env


`.env` 写入两行，换成你自己的域名和邮箱：

ini
DOMAIN=n8n.example.com
LETSENCRYPT_EMAIL=admin@example.com


4. 创建 `docker-compose.yaml`，n8n 和 Traefik 各一个服务。Traefik 监听 80/443 并强制 HTTP 跳转 HTTPS，通过 HTTP-01 验证自动签发 Let's Encrypt 证书；n8n 只在 Docker 内网暴露 5678 端口：

yaml
services:
  traefik:
    image: traefik:v3.6
    container_name: traefik
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=${LETSENCRYPT_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    restart: unless-stopped
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    hostname: n8n
    expose:
      - "5678"
    volumes:
      - "./n8n-data:/home/node/.n8n"
    environment:
      - N8N_HOST=${DOMAIN}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${DOMAIN}/
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
    restart: unless-stopped
volumes:
  letsencrypt:


5. 启动并验证：

bash
docker compose up -d
docker compose ps


两个容器都是 Up 状态、traefik 映射了 80 和 443，就说明起来了。有问题看 `docker compose logs`。

6. 在 DNS 服务商那里建好 A 记录，指向 Vultr 服务器的 IP，然后浏览器打开 `https://你的域名`。首次访问注册的账号就是 owner 账号，之后就能在节点编辑器里搭工作流或导入社区模板。

## 新手最容易踩的三个坑

大部分部署卡壳，都卡在下面这三处，每一条都有现成的解法。

**数据目录权限。** n8n 容器内以 `node` 用户（UID 1000）运行，挂载目录如果属主不对，容器写不进去，表现往往是启动报错或设置保存失败。官方文档专门用一步做 `sudo chown -R 1000:1000 n8n-data`，很多教程直接挂载目录就完事，读者卡在权限上报错半天。

**WEBHOOK_URL 与域名不一致。** 环境变量里的 `N8N_HOST` 和 `WEBHOOK_URL` 必须写成你实际访问的 https 域名，而且域名要提前解析到服务器 IP。webhook 收不到回调时，先查这两处；SSL 相关报错则检查 Vultr 防火墙设置里 80、443 端口是否放行，端口不通时证书申请也会失败。

**N8N_ENCRYPTION_KEY。** n8n 用这个密钥加密你保存的凭证。部署时就设一个随机字符串并保存好——丢了它，已存的凭证无法解密。走 Traefik 方案的话，把它加进 n8n 服务的 environment 列表即可。

## HTTPS 用 Traefik 还是 Nginx + Certbot

两条路线都能实现全自动 HTTPS，区别在于东西放在哪一层管理。

Traefik 是本文主线路线：路由规则写在 compose 的 labels 里，容器一启动就自动签发证书，HTTP 自动跳转 HTTPS，证书存在 `letsencrypt` 卷里，重启不丢。整套服务一个 `docker-compose.yaml` 管完，适合服务器上主要跑 Docker 的场景。

Nginx + Certbot 则是另一条被广泛验证的路：n8n 容器只绑定到 `127.0.0.1:5678`，Nginx 做反代，`certbot --nginx -d 你的域名` 一条命令完成证书申请并自动改好 Nginx 配置，之后每 90 天自动续期。服务器上已经用 Nginx 挂了别的站点时，这条路更顺。

用 Nginx 反代时，参考配置里 `proxy_buffering off`、`chunked_transfer_encoding off` 这几行别删，它们避免 webhook 的响应被 Nginx 缓冲。

## 备份与日常维护

数据全在挂载目录里：工作流、加密后的凭证、执行历史。最省事的备份是每天凌晨跑一次 tar 打包，保留最近 7 天：

bash
#!/bin/bash
tar -czf /opt/backups/n8n-$(date +%Y%m%d).tar.gz /opt/n8n/data
find /opt/backups -name 'n8n-*.tar.gz' -mtime +7 -delete


crontab 里加一行 `0 2 * * * /opt/n8n/backup.sh` 就行。compose 里的 `restart: unless-stopped` 负责容器崩溃或服务器重启后自动拉起，日常基本不用管。工作流多、执行频繁的负载，可以把数据从默认的 SQLite 换成 PostgreSQL，这是官方文档建议的生产级方向。

## 常见问题

**1GB 内存真的够跑 n8n 吗？**

起步够用。跑内存占用大的 JS 节点或 AI agent 时，2GB 是更稳的选择。

**没有域名能部署吗？**

想正常用 webhook 和 HTTPS，就需要域名：Let's Encrypt 的 HTTP-01 验证和 n8n 的 `WEBHOOK_URL` 都依赖域名解析到服务器 IP。先建好 A 记录再部署，能少走弯路。

**自托管一年到底花多少钱？**

$6 方案一年 $72，$12 方案一年 $144；n8n Cloud 同期是 $240–600。域名费用另算。👉 [查看 Vultr 各档套餐的当前价格](https://bit.ly/VulTR)

**数据都存在哪里？**

全在你服务器的挂载目录（`./n8n-data`）里，备份脚本打包的就是这个目录。工作流和凭证不会上传到任何第三方。

## 最后：按你的用途定方案

替自己省掉重复劳动、工作流几十个以内，$6 入门档就够，以后不够再升配；要跑 AI agent、重数据管道或给团队用，一步到位上 $12 的 2GB 方案，省得中途迁移。下单前确认两件事：域名 A 记录已经指到服务器 IP，Vultr 防火墙放行了 80 和 443 端口。这两处没问题，后面基本就是复制粘贴的事。👉 [开通 Vultr，把自动化搬出自己的订阅账单](https://bit.ly/VulTR)
