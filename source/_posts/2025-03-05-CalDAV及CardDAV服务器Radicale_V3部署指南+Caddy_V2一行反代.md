---
layout: post
cid: 24
title: CalDAV及CardDAV服务器Radicale V3部署指南+Caddy V2一行反代
slug: radicale-deploy
date: 2025/03/05 11:58:00
updated: 2025/03/05 16:02:41
status: publish
author: 神木友希
categories:
  - 主机折腾记录
tags:
  - caddy
  - radicale
desc: 
img:
---


[Baïkal](https://sabre.io/baikal/)的部署比较简单，然而由于某种神秘原因，Baïkal的日历在iOS上无法连接，即使我按照同类Github Issue中的解决方案在反代配置中使用了rewrite依然无法战胜，只好用回我的老朋友[Radicale](https://radicale.org)。
之前小鸡配置比较低，只跑得起这种超轻量级的服务，尽管现在在小鸡上部署了owncloud和nextcloud，我还是更习惯Radicale的管理界面，就有了今天这篇文章。
Radicale官方的文档写得还是很详细的，可以自行查阅。
**前置条件：以下操作在Debian 12 bookworm中使用root用户完成。**
## 安装步骤
安装Python3虚拟环境：
`root@debvmde:~# apt install python3-venv`
创建并进入radicale虚拟环境：
```
root@debvmde:~$ python3 -m venv radicale
root@debvmde:~$ radicale/bin/pip install pip --upgrade setuptools wheel
root@debvmde:~$ source radicale/bin/activate
```
这时可以看到`root@debvmde:~# `已经变成了`(radicale) root@debvmde:~# `，说明我们已经进入虚拟环境，可以pip install了。
运行一下：`pip install --upgrade radicale && radicale --version`
创建配置文件：`mkdir -p .config/radicale && nano .config/radicale/config`，根据官方文档的说法：
>Radicale tries to load configuration files from `/etc/radicale/config` and `~/.config/radicale/config`. Custom paths can be specified with the `--config /path/to/config` command line argument or the RADICALE_CONFIG environment variable. Multiple configuration files can be separated by : (resp. ; on Windows). Paths that start with ? are optional.

创建用户名和密码文件`.config/radicale/users`，可选的加密方式有plain/md5/bcrypt/autodetect，此处选用md5. （当然md5也并不是首选，关于怎么用htpasswd，请自行查阅官方文档）
我们希望选用的用户名和密码分别是user和password，如果采用plain加密，`.config/radicale/users`内就会存有明文的用户名和密码`user:password`，这样显然是不太安全的，所以需要以哈希值的形式存储密码。
使用htpasswd md5哈希生成器，把密码加密，得到形如`$apr1$j9JJ7hNw$aY8K5tLzVq4hi1XX/CnD70`的字符串，我们的密码文件中加入`user:$apr1$j9JJ7hNw$aY8K5tLzVq4hi1XX/CnD70`一行即可。
打开配置文件`~/.config/radicale/config`，加入以下内容：
```
[server]
hosts = 0.0.0.0:5232, [::]:5232

[storage]
filesystem_folder=/root/radicale/collections

[auth]
type = htpasswd
htpasswd_filename = /root/.config/radicale/users
htpasswd_encryption = md5

[server]
max_connections = 20
# 100 Megabyte
max_content_length = 100000000
# 30 seconds
timeout = 30

[auth]
# Average delay after failed login attempts in seconds
delay = 1
```
现在可以启动服务了：运行`radicale`。
访问`<serverip>:5232`，输入user和password，看看是否能成功进入。
创建service：`nano /etc/systemd/system/radicale.service`
加入以下内容：
```
[Unit]
Description=A simple CalDAV (calendar) and CardDAV (contact) server

[Service]
ExecStart=/root/radicale/bin/python /root/radicale/bin/radicale

Restart=on-failure

[Install]
WantedBy=default.target
```
启动：`systemctl enable radicale && systemctl start radicale`。
## 反代
Caddy的配置非常简单：
```
radicale.example.org {
	reverse_proxy localhost:5232
}
```

## 使用官方文档中的SHA-512对密码进行哈希时的注意事项
接下来是radicale官方文档的错误：
>The secure way
The users file can be created and managed with htpasswd:
>### Create a new htpasswd file with the user "user1" using SHA-512 as hash method
>`$ htpasswd -5 -c /path/to/users user1`
>`New password:`
>`Re-type new password:`
>### Add another user
>`$ htpasswd -5 /path/to/users user2`
>`New password:`
>`Re-type new password:`
>Authentication can be enabled with the following configuration:
>```
>[auth]
>type = htpasswd
>htpasswd_filename = /path/to/users
>htpasswd_encryption = autodetect #注意看注意看
>```

千万不要！用！autodetect！
是什么方法就写什么方法！比如我用SHA-512，就老老实实写`htpasswd_encryption = sha512`.
否则你会在`journalctl --unit radicale.service`收获一串美丽的报错：`radicale.service: Failed to determine user credentials: No such process`/`Failed at step USER spawning /root/radicale/bin/python: No such process`。


##参考文献：
https://radicale.org/v3.html
https://sigmdel.ca/michel/ha/aml912/radicale_en.html
https://www.atlantic.net/dedicated-server-hosting/how-to-install-radicale-calendar-caldav-and-carddav-on-ubuntu-20-04/