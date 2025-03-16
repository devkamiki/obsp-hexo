---
layout: post
cid: 5
title: docker搭建owncloud和onlyoffice并使用caddy反代，实现在线编辑文档
slug: 5
date: 2025/02/10 15:05:00
updated: 2025/02/10 18:23:21
status: publish
author: 神木友希
categories: 
  - 主机折腾记录
tags: 
  - docker
  - owncloud
  - onlyoffice
  - caddy
---


前置条件：Intel(R) Xeon(R) CPU E5-2690 v4 @ 2.60GHz 1 Virtual Core；Debian 12 bookworm；Caddy v2；docker 和 docker compose
owncloud使用docker compose部署，首先创建一个你自己喜欢的目录，等下我们把这个docker容器中的文件映射到这个目录。命令：`mkdir whatever/path/to/owncloud && cd whatever/path/to/owncloud`
编辑一下dockerfile：`nano docker-compose.yml`，模板来自[官方](https://doc.owncloud.com/server/next/admin_manual/installation/docker/#docker-compose)：
```
  volumes:
    files:
      driver: local
    mysql:
      driver: local
    redis:
      driver: local
  
  services:
    owncloud:
      image: owncloud/server:latest
      container_name: owncloud_server
      restart: always
      ports:
        - 8080:8080 #左边改成你自己喜欢的端口
      depends_on:
        - mariadb
        - redis
      environment:
        - OWNCLOUD_DOMAIN=owncloud.mydomain.org
        - OWNCLOUD_TRUSTED_DOMAINS=owncloud.mydomain.org,my_ip_address 
        #注意，如果要添加多个信任域名/ip，用半角逗号分隔，不要加空格，不要加端口号。
        - OWNCLOUD_DB_TYPE=mysql
              - OWNCLOUD_DB_NAME=owncloud
        - OWNCLOUD_DB_USERNAME=owncloud
        - OWNCLOUD_DB_PASSWORD=owncloud
        - OWNCLOUD_DB_HOST=mariadb
        - OWNCLOUD_ADMIN_USERNAME=admin
        - OWNCLOUD_ADMIN_PASSWORD=Th1sPa$$wordIs$uper$ecret!
        - OWNCLOUD_MYSQL_UTF8MB4=true
        - OWNCLOUD_REDIS_ENABLED=true
        - OWNCLOUD_REDIS_HOST=redis
      healthcheck:
        test: ["CMD", "/usr/bin/healthcheck"]
        interval: 30s
        timeout: 10s
        retries: 5
      volumes:
        - files:/mnt/data
            
    mariadb:
      image: mariadb:10.11 # minimum required ownCloud version is 10.9
      container_name: owncloud_mariadb
          restart: always
      environment:
        - MYSQL_ROOT_PASSWORD=owncloud
        - MYSQL_USER=owncloud
        - MYSQL_PASSWORD=owncloud
        - MYSQL_DATABASE=owncloud
        - MARIADB_AUTO_UPGRADE=1
      command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
      healthcheck:
        test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=owncloud"]
        interval: 10s
        timeout: 5s
        retries: 5
      volumes:
        - mysql:/var/lib/mysql
  
    redis:
      image: redis:6
      container_name: owncloud_redis
      restart: always
      command: ["--databases", "1"]
      healthcheck:
        test: ["CMD", "redis-cli", "ping"]
        interval: 10s
        timeout: 5s
        retries: 5
      volumes:
        - redis:/data
  
```
然后 docker 启动！ `docker compose up -d`
接下来就是把域名owncloud.mydomain.org解析到服务器ip地址，（这个时候编辑caddyfile并重启caddy也可以，不过我们装完onlyoffice再一起改）等下的onlyoffice也需要一个域名，所以解析的时候顺便把office.mydomain.org也解析到服务器的ip地址吧。
可以使用`docker ps`查看owncloud和依赖有没有正常运行，如果不行，看一下`docker compose logs -f`，排查原因。如果行，登录你刚刚在docker-compose.yml里面设置的owncloud管理员账户，进行基础设置，然后到owncloud的应用市场market里面下载onlyoffice这个应用。
owncloud设置大功告成，可以开始设置onlyoffice了：
我们在你喜欢的端口（比如12345）上面跑一个onlyoffice的docker 容器：
```
docker run -i -t -d -p 12345:80 --restart=always onlyoffice/documentserver
```
这个时候运行一下`docker ps`，查看刚刚创建的这个容器id，然后运行：
```
  docker exec <onlyoffice-container-id> /var/www/onlyoffice/documentserver/npm/json -f /etc/onlyoffice/documentserver/local.json '
  - services.CoAuthoring.secret.session.string'
```
可能就有人要问了：这个命令是干嘛的呢？
运行之后返回的字符串是我们onlyoffice的JWT secret key，等下在owncloud里面设置要用。把这个字符串记下来，下文中我们会用<JWT_secret>指代它。
好了，现在我们来设置Caddy吧。之所以在这一步设置Caddy，是因为onlyoffice（以及collabora online）要求服务端和客户端使用同样的协议，比如同为https或者同为http。我们使用Caddy进行反代，Caddy会自动给域名申请证书，这样owncloud客户端和onlyoffice服务端都是https协议的了。
如果你用的是nginx或者apache，也是一样的，如果需要https，请给两个域名手动申请证书，这里就不详细展开了。
我们编辑一下caddyfile吧：`nano /etc/caddy/Caddyfile`
加上下面这两行：
```
  owncloud.mydomain.org {
  	reverse_proxy localhost:8080
  }
  
  office.mydomain.org {
  	reverse_proxy localhost:12345
  }
```
保存并退出，重载一下caddy：`systemctl reload caddy`
现在通过owncloud.mydomain.org可以以https访问我们的owncloud实例了。
进入owncloud设置，左菜单栏下滑到底部选择“额外的”，在这里你可以设置onlyoffice服务器。
我们在ONLYOFFICE Docs地址那里填入office.mydomain.org，在秘钥那里填入<JWT_secret>。
点击保存，大功告成！
尝试在owncloud文件中打开一个.docx结尾的文档，你会看到onlyoffice加载并启动。
好了，结束了！

吐槽一下：
这篇文章是在Logseq上面写的。Logseq也太难用了吧，这个块状排版真的不适合习惯普通段落式的人……尤其是代码块难用得蛋疼，一旦把一个块变为代码块就无法回到文本编辑模式了……
Logseq的多端同步也很糟糕，使用git自动commit让我不得不每分钟看见一次错误信息，而且iOS的git客户端Working Copy也比较贵。目前我的做法是Syncthing 在 Android手机和Linux PC之间同步，38块钱买一个iOS的Syncthing客户端Möbius Sync高级版（当然由于iOS的后台机制，功能是残废的）把iPad也加入同步链。Syncthing没有中心服务器，我又不能保证有一台设备24小时运行且可访问（这个问题在我整了台小主机之后解决了。我要试试用小主机当中心服务器看看是否改善），所以经常在同步后出现文件冲突，非常烦人。
iOS用户需要注意：Logseq只能打开本地文件夹，而且是在Logseq的iOS应用沙盒中的文件夹，这个问题没办法通过挂载云存储解决。要想和其他设备同步，只能在你的同步软件上面下功夫，让同步软件能够访问Logseq的应用沙盒，Möbius Sync如此，Working Copy应该也是如此（没用过）。