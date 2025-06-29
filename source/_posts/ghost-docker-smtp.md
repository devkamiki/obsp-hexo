---
title: docker 部署 Ghost 博客，并自定义推送邮件服务（以 Resend 为例）
date: 2025-06-29 13:03:23
tags:
  - docker
  - ghost
  - blog
  - resend
  - smtp
categories:
  - 日常
---
## 前言

Ghost 是一个美观、易用、可以自托管的开源博客平台。

与静态站点生成器（Hugo, Hexo, Jekyll, Astro, Nuxt 等）不同，Ghost 更像 WordPress 和 Typecho，需要独立的数据库，并且具有更为丰富的功能。Ghost 提供了一个管理后台，不仅可以实时保存博客文稿的写作进度，随时进行创作、编辑、发表，还能进行用户权限设置，为一个站点建立多个管理员及创作者用户。Ghost 集成了文章数据分析和触达读者的推送邮件服务，不需要单独设置定时任务或邮件内容。对于需要提供增值服务的博客，Ghost 内置的订阅功能可以识别用户的订阅状态，提供 "Subscriber Only" 的博文内容等，为专业的博客维护者提供了便利。著名的 Linux 入门博客 [It's FOSS](https://itsfoss.com) 就是一个 Ghost 实例，如果你曾经订阅 It's FOSS 的新闻邮件，或是注册账户在上面进行评论，那么你可能会注意到， It's FOSS 提供了 $24 / 年的会员订阅，在登录时，会员不会在网站上看到广告。

由于 Ghost 的 docker 镜像是由社区开发者维护的，没有官方文档指导，我在安装的过程中遇到了一些波折，尤其是设置推送邮件服务。

## 安装和设置

我采用的是 [docker 镜像](https://hub.docker.com/_/ghost/)维护者在文档中建议的 compose.yml，在此基础上进行了小小的改动：

```yaml

services:

  ghost:
    image: ghost:5-alpine
    restart: always
    ports:
      - 127.0.0.1:2368:2368 # 改成你喜欢的端口
    environment:
      # see https://ghost.org/docs/config/#configuration-options
      database__client: mysql
      database__connection__host: db
      database__connection__user: root
      database__connection__password: ChangeItToASecretPassword
      database__connection__database: ghost
      # this url value is just an example, and is likely wrong for your environment!
      url: https://example.org
      # contrary to the default mentioned in the linked documentation, this image defaults to NODE_ENV=production (so development mode needs to be explicitly specified if desired)
      #NODE_ENV: development
    volumes:
      - ./ghost-content:/var/lib/ghost/content
      # 这里我改了映射路径，方便打包备份，
      # 如果你更喜欢 ghost:/var/lib/ghost/content 也可以自己改
      - ./config/config.production.json:/var/lib/ghost/config.production.json
      # 目的是配置邮件推送
  db:
    image: mariadb:10.6
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ChangeItToASecretPassword 
      # 和上面那个 database__connection__password 要相同哦
    volumes:
      - ./db:/var/lib/mysql

volumes:
  ghost:
  db:

```

Ghost 自托管版默认通过 Mailgun 发送订阅邮件，可以直接在后台设置域名和 Mailgun API，这是唯一支持的服务提供商。当你使用 docker compose 安装并且没有指定 Mailgun 配置的时候，会尝试从本机以 `noreply@example.org` 的地址向注册用户和订阅者发送邮件。当然，如果服务器的 25 端口关闭，或域名没有配置合适的 spf dkim dmarc 记录，这封邮件很可能不会进入读者的收件箱。因此，我们需要编辑 docker 容器内的 `/var/lib/ghost/config.production.json`，让它使用合适的 SMTP 服务进行邮件推送。接下来我们会以 Resend 为例，进行配置。

先登录 Resend 后台，添加域名，这里我们使用 `example.org` .

添加 DNS 记录，看起来应该像是这样：

| Type | Name              | Content                               | TTL  | Priority |
| ---- | ----------------- | ------------------------------------- | ---- | -------- |
| MX   | send              | feedback-smtp.eu-west-1.amazonses.com | Auto | 10       |
| TXT  | send              | v=spf1 include:amazonses.com ~all     | Auto |          |
| TXT  | resend._domainkey | p=一长串字符                               | Auto |          |
| TXT  | _dmarc            | v=DMARC1; p=none;                     | Auto |          |

当所有 DNS 记录验证完毕后，就可以选择 API Keys，获取 API 密钥了。你可以为每个用途单独创建一个 API 密钥，我们将刚刚创建的密钥命名为 Ghost，方便管理。

还记得刚刚我们映射出了配置文件吗？找到它，并且调整为如下的样子：

```json
{
  "url": "http://localhost:2368",
  "server": {
    "port": 2368,
    "host": "0.0.0.0"
  },
  "mail": {
    "transport": "SMTP",
    "options": {
        "service": "Resend",
        "host": "smtp.resend.com",
        "port": 587,
        "secureConnection": true,
        "auth": {
            "user": "resend",
            "pass": "YourApiKey"
        }
    }
  },
  "logging": {
    "path": "content/logs/",
    "level": "error",
    "rotation": {
      "enabled": true,
      "count": 10,
      "period": "1d"
  },
    "transports": [
      "file",
      "stdout"
    ]
  },
  "process": "systemd",
  "paths": {
    "contentPath": "/var/lib/ghost/content"
  }
}

```

这样就大功告成了！所有的订阅邮件将通过 Resend 的 SMTP 服务器传送。由于你授权了该服务器使用 `@example.org`，产生的邮件将不会因为域名的 spf dkim dmarc 记录进入垃圾箱。

## 后记

Ghost 很美好，很便捷，也很惊艳——但是我最后还是选择了 Hexo，为什么呢？

我想，这大概和我的不安全感有关——免费 PaaS 和数据库不是很多，并且每次重启都会清空数据。虽然我买了服务器，自己搭建过 Typecho，设置了定时备份，但我还是害怕有一天会因为自己鲁莽的行为丢失博客的配置。我也害怕自己会有一天因为交不起几欧一个月的服务器费用，被迫将站点关停。而能免费托管静态站点的平台很多，Netlify, Statichost.eu, Codeberg Pages, SourceHut Pages... 数据的存储也无须过于担心，使用 GitHub 或 GitLab 仓库存储所有的站点内容，搬家也很轻松。

更重要的是，我更欣赏静态站点生成器的轻量和自由——可以接入 Giscus 等外部评论区，不强制读者在站点上注册才能进行评论；可以直接撰写 Markdown 格式的博文，导入和导出更加方便。阻碍我转向 Ghost 的一大原因正是我无法通过后台进行 Markdown 文章的导入，官方只提供了对 WordPress，Medium 等平台的支持，我试图通过魔改 Jekyll 迁移插件实现支持 Hexo 的效果未果。由于实在太麻烦，在经历了五秒的思想斗争后，我放弃了逐项导入所有博文的想法，决定 `docker compose down` 逃之夭夭。

还有，Ghost 的核心功能——会员制度，对我来说确实没什么用。我不关心博文的变现和推广，目标只有一个：当潜在读者使用搜索引擎敲下问题时，他们可能在这里找到答案。

