---
layout: post
cid: 7
title: docker compose部署订阅管理软件wallos，并启用telegram机器人通知
slug: wallos
date: 2025/02/11 00:48:00
updated: 2025/02/13 15:28:56
status: publish
author: 神木友希
categories: 
  - 主机折腾记录
tags: 
  - docker
  - caddy
  - Wallos
desc: 
img: 
---


话费，域名，邮箱，VPS，密码管理器，云存储，流媒体会员……数字生活的订阅开支无处不在，却难以管理。你是否也有需要的服务到期忘记续费，而不需要的服务不小心自动续费的烦恼? 别担心，已经有开发者推出了解决方案，那就是订阅管理软件[Wallos](https://github.com/ellite/Wallos)。这是一个可以自部署的开源项目，不仅有多种货币和支付方式，还有各种各样的提醒方式，包括email, discord, pushover, telegram, gotify和webhooks。项目部署支持baremetal和docker，无论你是想高度自定义，还是想简便快捷，都可以做到。
**这篇文章聚焦使用docker compose的方式部署Wallos管理订阅，然后使用tg机器人通知来实现提醒。**
## Wallos的部署
我们先`mkdir wallos && cd wallos`
编辑一下`nano docker-compose.yaml`，添加如下内容:
```
services:
  wallos:
    container_name: wallos
    image: bellamy/wallos:latest
    ports:
      - "8282:80/tcp"#左边改成你喜欢的端口
    environment:
      TZ: 'America/Toronto'#改成你所在的时区
    # Volumes store your data between container upgrades
    volumes:
      - './db:/var/www/html/db'
      - './logos:/var/www/html/images/uploads/logos'
    restart: unless-stopped
```
最后，`docker compose up -d`，大功告成!
运行`docker compose logs -f`看看是否正常启动。
## 解析域名，设置反向代理
把域名wallos.mydomain.org解析到服务器ip地址。
这里用的是Caddy V2，Apache和Nginx用户请自行同理可得。
`nano /etc/caddy/Caddyfile`
加入以下内容:
```
wallos.mydomain.org {
    reverse_proxy localhost:8282
}
```
重载一下Caddy: `systemctl reload caddy`
打开浏览器访问wallos.mydomain.org，看看是否正常。创建管理员账号，设置首选货币等，此处不再赘述。
## Telegram机器人绑定
进入Wallos，点击头像，点击[设置]，点击[通知]中的[Telegram]。
打开Telegram，向@BotFather发送/newbot，按照提示操作。
创建完成后，BotFather会发送一条信息:
```
Done! Congratulations on your new bot. You will find it at t.me/<botname›. You can now add a description, about section and profile picture for your bot, see /help for a list of commands. By the way, when you've finished creating your cool bot, ping our Bot Support if you want a better username for it. Just make sure the bot is fully operational before you do this.

Use this token to access the HTTP API:‹yourapi›
Keep your token secure and store it safely, it can be used by anyone to control your bot.

For a description of the Bot API, see this page: https://core.telegram.org/bots/api
```
把获得的`<yourapi>`填入Wallos Telegram通知的第一栏。
浏览器访问`https://api.telegram.org/bot<yourapi>/getUpdates`，回到tg点击`t.me/<botname>`给你的机器人发条消息，然后浏览器刷新，即可获得一个JSON文件，其中键"id"的值就是你的chat id，把这个值填入Wallos Telegram通知的第二行，勾选启用，点击测试。
如果tg机器人给你发送了消息，说明配置成功运行。接下来只要为订阅设置通知日期，就可以静待Wallos的自动提醒了!