---
tags:
  - docker
  - e2ee
  - OSS
  - Notesnook
title: 使用docker自建e2ee开源笔记软件Notesnook服务器
categories:
  - 主机折腾记录
date: 2025-04-03 11:57:01
---
2025.04.11 更新：邮件发送地址可以通过在 `.env` 中指定 `NOTESNOOK_SENDER_EMAIL` 的值来确定，这样就解决了会使用 SMTP 登录名作为发件人地址的问题

## 前言

笔记软件的类型非常多，划分的标准也不尽相同。以笔记的主要存储位置为标准，既有注重云端的 Notion, OneNote 和 FlowUs 等，也有本地优先的 Obsidian, Logseq 和思源笔记等；以编辑器类型为标准，既有专注于纯文本 / 富文本 / Markdown 编辑的，也有八面玲珑，兼顾日历、任务、书签管理功能的；以加密程度为标准，既有注重隐私安全的，也有注重便捷分享的。当一个笔记软件找到自己的定位，获得了一些目标人群的认可时，也往往会顾此失彼，让另一些潜在客户悄悄流失。

例如，Obsidian 以强大的文本编辑能力、本地优先，无须网络或登录账户的快速便捷性和直接使用 .md 文档作为存储格式的开放性受到了用户的交口称赞，与此同时也因为同步的不便遭到了诟病；强调隐私保护的Standard Notes，因简洁的界面、开放源代码和经第三方审计的安全性在私人笔记市场受到欢迎，但与昂贵的订阅价格无法匹配的基础功能也令用户怨声载道。

不论如何，这些赫赫有名的笔记软件都拥有自己的独门绝技。在这个硬件产品逐渐同质化的时代，软件产品的差异化，能让市场更加百花齐放，满足不同客户的需求。

今天我们尝试部署的笔记软件 Notesnook 就是一个很好的例子。Notesnook 是由巴基斯坦的一个 3 人小公司 Streetwriters 开发的，在 2021 年发布了第一个版本，并在 2022 年以 GPL-3.0 许可证发布了服务端与客户端的源代码。同样以“端到端加密”的特性作为主打卖点，与拥有 8 年历史的 Standard Notes 相比，Notesnook 是一个年轻的竞争者。与 Standard Notes 较为稳健、保守的风格不同，Notesnook 广泛地听取用户意见，积极地引入各种新功能，而它比 Standard Notes 低许多的高级版定价以及相对慷慨的免费层服务吸引了不少 Standard Notes 的忠实用户。不少用户表示，“阻碍我转向 Notesnook 的唯一障碍就是不能自托管”。随着自托管进入 Alpha 阶段和服务端 Docker 镜像的发布，“不能自托管”的时代也已经结束了。尽管官方的完整版自托管文档尚未发布，但是已经有不少用户使用了 GitHub 仓库中的 Building From Source 或 `docker compose` 成功进行了部署。

我们使用 `docker compose` 进行简便部署，并且用 Caddy 进行反代。

## 前置条件

- docker 和 docker compose
- 4 个域名或者子域名，下面用 `notes.example.io, mono.example.io, events.example.io, auth.example.io` 指代
- Caddy V2

## 设置 compose.yml 与 .env

官方提供了一个开箱即用的 `docker-compose.yml` ：

`wget https://raw.githubusercontent.com/streetwriters/notesnook-sync-server/master/docker-compose.yml`

大部分内容我们都是不需要修改的：

```yaml
x-server-discovery: &server-discovery
  NOTESNOOK_SERVER_PORT: 5264
  NOTESNOOK_SERVER_HOST: notesnook-server
  IDENTITY_SERVER_PORT: 8264
  IDENTITY_SERVER_HOST: identity-server
  SSE_SERVER_PORT: 7264
  SSE_SERVER_HOST: sse-server
  SELF_HOSTED: 1
  IDENTITY_SERVER_URL: ${AUTH_SERVER_PUBLIC_URL}
  NOTESNOOK_APP_HOST: ${NOTESNOOK_APP_PUBLIC_URL}

x-env-files: &env-files
  - .env

services:
  validate:
    image: vandot/alpine-bash
    entrypoint: /bin/bash
    env_file: *env-files
    command:
      - -c
      - |
        # List of required environment variables
        required_vars=(
          "INSTANCE_NAME"
          "NOTESNOOK_API_SECRET"
          "DISABLE_SIGNUPS"
          "SMTP_USERNAME"
          "SMTP_PASSWORD"
          "SMTP_HOST"
          "SMTP_PORT"
          "AUTH_SERVER_PUBLIC_URL"
          "NOTESNOOK_APP_PUBLIC_URL"
          "MONOGRAPH_PUBLIC_URL"
          "ATTACHMENTS_SERVER_PUBLIC_URL"
        )

        # Check each required environment variable
        for var in "$${required_vars[@]}"; do
          if [ -z "$${!var}" ]; then
            echo "Error: Required environment variable $$var is not set."
            exit 1
          fi
        done

        echo "All required environment variables are set."
    # Ensure the validate service runs first
    restart: "no"

  notesnook-db:
    image: mongo:7.0.12
    hostname: notesnook-db
    volumes:
      - dbdata:/data/db
      - dbdata:/data/configdb
    networks:
      - notesnook
    command: --replSet rs0 --bind_ip_all
    depends_on:
      validate:
        condition: service_completed_successfully
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh mongodb://localhost:27017 --quiet
      interval: 40s
      timeout: 30s
      retries: 3
      start_period: 60s
  # the notesnook sync server requires transactions which only work
  # with a MongoDB replica set.
  # This job just runs `rs.initiate()` on our mongodb instance
  # upgrading it to a replica set. This is only required once but we running
  # it multiple times is no issue.
  initiate-rs0:
    image: mongo:7.0.12
    networks:
      - notesnook
    depends_on:
      - notesnook-db
    entrypoint: /bin/sh
    command:
      - -c
      - |
        mongosh mongodb://notesnook-db:27017 <<EOF
          rs.initiate();
          rs.status();
        EOF

  notesnook-s3:
    image: minio/minio:RELEASE.2024-07-29T22-14-52Z
    ports:
      - 9000:9000
    networks:
      - notesnook
    volumes:
      - s3data:/data/s3
    environment:
      MINIO_BROWSER: "on"
    depends_on:
      validate:
        condition: service_completed_successfully
    env_file: *env-files
    command: server /data/s3 --console-address :9090
    healthcheck:
      test: timeout 5s bash -c ':> /dev/tcp/127.0.0.1/9000' || exit 1
      interval: 40s
      timeout: 30s
      retries: 3
      start_period: 60s

  # There's no way to specify a default bucket in Minio so we have to
  # set it up ourselves.
  setup-s3:
    image: minio/mc:RELEASE.2024-07-26T13-08-44Z
    depends_on:
      - notesnook-s3
    networks:
      - notesnook
    entrypoint: /bin/bash
    env_file: *env-files
    command:
      - -c
      - |
        until mc alias set minio http://notesnook-s3:9000 ${MINIO_ROOT_USER:-minioadmin} ${MINIO_ROOT_PASSWORD:-minioadmin}; do
          sleep 1;
        done;
        mc mb minio/attachments -p

  identity-server:
    image: streetwriters/identity:latest
    ports:
      - 8264:8264
    networks:
      - notesnook
    env_file: *env-files
    depends_on:
      - notesnook-db
    healthcheck:
      test: wget --tries=1 -nv -q  http://localhost:8264/health -O- || exit 1
      interval: 40s
      timeout: 30s
      retries: 3
      start_period: 60s
    environment:
      <<: *server-discovery
      MONGODB_CONNECTION_STRING: mongodb://notesnook-db:27017/identity?replSet=rs0
      MONGODB_DATABASE_NAME: identity

  notesnook-server:
    image: streetwriters/notesnook-sync:latest
    ports:
      - 5264:5264
    networks:
      - notesnook
    env_file: *env-files
    depends_on:
      - notesnook-s3
      - setup-s3
      - identity-server
    healthcheck:
      test: wget --tries=1 -nv -q  http://localhost:5264/health -O- || exit 1
      interval: 40s
      timeout: 30s
      retries: 3
      start_period: 60s
    environment:
      <<: *server-discovery
      MONGODB_CONNECTION_STRING: mongodb://notesnook-db:27017/?replSet=rs0
      MONGODB_DATABASE_NAME: notesnook
      S3_INTERNAL_SERVICE_URL: "http://notesnook-s3:9000"
      S3_INTERNAL_BUCKET_NAME: "attachments"
      S3_ACCESS_KEY_ID: "${MINIO_ROOT_USER:-minioadmin}"
      S3_ACCESS_KEY: "${MINIO_ROOT_PASSWORD:-minioadmin}"
      S3_SERVICE_URL: "${ATTACHMENTS_SERVER_PUBLIC_URL}"
      S3_REGION: "us-east-1"
      S3_BUCKET_NAME: "attachments"

  sse-server:
    image: streetwriters/sse:latest
    ports:
      - 7264:7264
    env_file: *env-files
    depends_on:
      - identity-server
      - notesnook-server
    networks:
      - notesnook
    healthcheck:
      test: wget --tries=1 -nv -q  http://localhost:7264/health -O- || exit 1
      interval: 40s
      timeout: 30s
      retries: 3
      start_period: 60s
    environment:
      <<: *server-discovery

  monograph-server:
    image: streetwriters/monograph:latest
    ports:
      - 6264:3000
    env_file: *env-files
    depends_on:
      - notesnook-server
    networks:
      - notesnook
    healthcheck:
      test: wget --tries=1 -nv -q  http://localhost:3000/api/health -O- || exit 1
      interval: 40s
      timeout: 30s
      retries: 3
      start_period: 60s
    environment:
      <<: *server-discovery
      API_HOST: http://notesnook-server:5264
      PUBLIC_URL: ${MONOGRAPH_PUBLIC_URL}

  autoheal:
    image: willfarrell/autoheal:latest
    tty: true
    restart: always
    environment:
      - AUTOHEAL_INTERVAL=60
      - AUTOHEAL_START_PERIOD=300
      - AUTOHEAL_DEFAULT_STOP_TIMEOUT=10
    depends_on:
      validate:
        condition: service_completed_successfully
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
networks:
  notesnook:

volumes:
  dbdata:
  s3data:
```

在这个文件中，我们可以看到，Notesnook 服务器的必需组件包括：
- Vandot
- MongoDB
- S3 存储，这里采用了自建 MinIO 实例的方式（应该和 Ente 一样，不是必须使用 MinIO，可以接入外部 S3，不过我没试过，这里先不演示）
- 身份验证服务器
- 同步服务器
- SSE，用于提醒事件
- Monograph，用于分享笔记的公开链接（可以加密）。

其中需要映射到的宿主机端口是 5264（同步服务器），6264（Monograph），7264（SSE）和 8264（身份服务器）。如果这些端口被占用，你可以选择映射到其他的端口。

我们需要进行个性化设置的是 `.env` 文件，官方同样提供了一个示例：

```bash
INSTANCE_NAME=self-hosted-notesnook-instance 
# 改成你的实例名称


NOTESNOOK_API_SECRET= 
# 随机生成一串 >32 位的长字符串


DISABLE_SIGNUPS=false
# 这个变量目前（2025年4月3日，v1.0-beta.1）还不能起效，
# 如果你需要禁用注册功能，请在自己注册完之后添加以下变量：
# DISABLE_ACCOUNT_CREATION=1


# SMTP 配置，用于接收 2FA 代码和重设密码，可以使用你自己的邮件提供商的 SMTP 配置


SMTP_USERNAME=
# 大部分情况下，是你的邮箱用户名，例如 user@example.org 
# 当 SMTP 用户名不是邮箱用户名的时候，会有 bug 出现，
# Notesnook 会尝试把发件人地址（也就是邮件的 From） 设置为 SMTP 用户名，这在某些情况下会导致发件被拒绝。
# 我没有找到单独设置发件人地址的方法，已经提交 Issue

SMTP_PASSWORD=
# SMTP 密码，根据你的邮件提供商的指引设置

SMTP_HOST=
# 例如 smtp.gmail.com

SMTP_PORT=
# SMTP 端口，例如 465

TWILIO_ACCOUNT_SID=
# 不需要设置

TWILIO_AUTH_TOKEN=
# 不需要设置

TWILIO_SERVICE_SID=
# 不需要设置

NOTESNOOK_CORS_ORIGINS=
# 不需要设置

NOTESNOOK_APP_PUBLIC_URL=https://app.notesnook.com
# 换成你自己的 APP 域名，如 https://notes.example.io

MONOGRAPH_PUBLIC_URL=http://localhost:6264
# 换成你的 Monograph 域名，如 https://mono.example.io

AUTH_SERVER_PUBLIC_URL=http://localhost:8264
# 换成你的身份服务器域名 https://auth.example.io


ATTACHMENTS_SERVER_PUBLIC_URL=http://localhost:9000
# MinIO 地址和端口，可以不用域名


MINIO_ROOT_USER=
# 不需要设置

MINIO_ROOT_PASSWORD=
# 不需要设置

```

设置完直接启动即可：`docker compose up -d`

## 反向代理配置

Nginx 和 Certbot 的配置参考这里：https://github.com/streetwriters/notesnook-sync-server/issues/20#issuecomment-2603896363

```nginx
server {
    listen 80;
    server_name auth.domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name auth.domain.com;

    ssl_certificate /etc/letsencrypt/live/auth.domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/auth.domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8264;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Notes Server - With WebSocket
server {
    listen 80;
    server_name notes.domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name notes.domain.com;

    ssl_certificate /etc/letsencrypt/live/notes.domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notes.domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:5264;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}

# Events Server - With WebSocket
server {
    listen 80;
    server_name events.domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name events.domain.com;

    ssl_certificate /etc/letsencrypt/live/events.domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/events.domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:7264;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}

# Monograph Server - With Cache
server {
    listen 80;
    server_name mono.domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name mono.domain.com;

    ssl_certificate /etc/letsencrypt/live/mono.domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mono.domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:6264;
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        proxy_cache_valid 200 60m;
        add_header X-Cache-Status $upstream_cache_status;
        expires 1h;
        add_header Cache-Control "public, no-transform";
    }
}
```

Caddy 的配置参考这里：https://github.com/streetwriters/notesnook-sync-server/issues/20#issuecomment-2763248500

```nginx
notes.example.io {
        reverse_proxy localhost:5264
}

mono.example.io {
        reverse_proxy localhost:6264
}

events.example.io {
        reverse_proxy localhost:7264
}

auth.example.io {
        reverse_proxy localhost:8264
}
```

正常情况下访问 `https://mono.example.io` 能看到 Monograph 的默认页面就成功了。

## 在 Notesnook 客户端自定义服务器

下载 Notesnook 的客户端，进入 Settings -> Servers，填写以下端点：

Sync server URL: `https://notes.example.io`      
Auth server URL: `https://auth.example.io`    
Events server URL: `https://events.example.io`     
Monograph server URL: `https:mono.example.io`  

然后就可以输入你的邮箱注册账户了~

需要注意的是，与 Vikunja 和 Vaultwarden 不同，Notesnook 的邮件验证不是可选项，所以需要填写可以接收到邮件的地址。

如果需要关闭注册，可以在自己注册完后修改 `.env` ，加入 `DISABLE_ACCOUNT_CREATION=1` 。