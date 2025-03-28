---
layout: post
cid: 20
title: docker自建Bitwarden的社区精简高级版Vaultwarden，并使用Caddy反代
slug: self-hosting-vaultwarden
date: 2025/02/19 20:44:00
updated: 2025/02/19 20:44:24
status: publish
author: 神木友希
categories: 
  - 主机折腾记录
tags: 
  - docker
  - caddy
  - vaultwarden
desc: 
img: 
---


在体验过市面上一圈密码管理器（第三方托管+自建）之后，我还是可耻地选择了商业，选择每个月掏2刀给Proton让它赚=。= 不过我是因为太懒&对密码管理的需求太复杂才做出如此选择的，大部分热爱折腾的mjj还是喜欢Keepass和Vaultwarden，今天就来讲一下小鸡自建Vaultwarden的配置。

## 启动 Docker 容器

先mk个dir ：`mkdir vw && cd vw`

`docker run`比较快，不过我个人觉得`docker compose`方便一些，`nano docker-compose.yaml`:

  ```
  services:
    bitwarden:
      image: vaultwarden/server:latest
      container_name: bitwarden
      restart: unless-stopped
      environment:
        - DOMAIN=https://vw.mydomain.org/asuperrandomstring
        - SIGNUPS_ALLOWED=true
        - WEB_VAULT_ENABLED=true
      volumes:
        - ./vw-data:/data
      ports:
        - "8080:80"
  
  ```

记得把域名的A记录加上，`vw.mydomain.org`指向Vaultwarden服务器的ip。

这里的path`/asuperrandomstring`可以拿密码管理器生成一个，然后写在纸上或者文本文档里。千万不要只放在你建好的Vaultwarden密码库里，原因你懂的，就像不要给2FA验证器加2FA登录验证一样……

## 配置`Caddyfile`:

  ```
  vw.mydomain.org {
   route {
          reverse_proxy /asuperrandomstring/* localhost:8080 {
              header_up X-Real-IP {http.request.header.Cf-Connecting-Ip}
          }
  }
  ```

配置好一起启动：`docker compose up -d && systemctl reload caddy`

访问`https://vw.mydomain.org/asuperrandomstring/`，创建账号，导入密码，不再赘述。

创建完所有需要的账号之后，记得关闭访客注册~

关闭访客注册的方式是`docker compose down`，然后修改`docker-compose.yaml`中`SIGNUPS_ALLOWED`的值为`false`，再`docker compose up -d`即可。
## 数据备份

为防止数据丢失，可以用之前提到过的[使用Filen CLI定时备份服务器数据](https://obsp.de/index.php/archives/filen-cli-backup.html)备份你的vw-data文件夹~如果这么做，确保你的Filen账号和密码是自己记得的，而不是存储在Vaultwarden保险库里面的，否则一旦服务器上的数据受损，想从Filen取回备份的时候，会把自己反锁在外面。