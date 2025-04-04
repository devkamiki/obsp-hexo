---
title: Caddy 使用 Buypass Go SSL 作为证书提供商
tags:
  - caddy
  - ssl
  - buypass
  - certbot
categories:
  - 日常
date: 2025-03-31 17:55:51
---

## 前言

大家好啊，今天教大家一个脱裤子放屁的 Caddy 小技巧。

由 Let's Encrypt 获取的 SSL 证书一般有效期为 90 天。虽然可以采用 Certbot 和 acme.sh  一类的脚本，或者交给 Caddy 自动续期，但可能还是有些小伙伴想要获取有效期更长的免费证书。

[Buypass Go SSL/TLS](https://www.buypass.com/products/tls-ssl-certificates/go-ssl) 是一家挪威证书签发商的产品，基于 ACME ，证书的有效期为 180 天。那么，如何在 Caddy 中使用它替换默认的 Let's Encrypt 呢？

## 自定义兼容 ACME 的 CA 的通用语法

在没有特别指明的情况下，Caddy 通常会使用 Let's Encrypt 或者 ZeroSSL 来作为 Certificate Authority。你也可以自定义 CA，这里有两种语法，可用于兼容 ACME 的签发者：

- 第一种，放在 `Caddyfile` 开头，作用于全局：
```nginx
{
	acme_ca https://acme.zerossl.com/v2/DV90 # 换成自定义 CA
	email hi@example.dev
}
```

- 第二种，放在每个域名的大括号里面，作用于域名： 
```nginx
tls hi@example.dev {
    ca https://acme.zerossl.com/v2/DV90 # 换成自定义 CA
    }
```

很可惜的是，经过我的尝试，这两种语法下 Buypass Go SSL 都无法获取到证书。大概是因为 Buypass 在某次更新后添加了 ARI 支持，导致了 Caddy 的 `panic:certificate worker: runtime error: invalid memory address`。

解决方法有两个：一是使用 Certbot 进行注册，再手动设置 Caddy 读取获得的证书；二是使用最新的 `acmez`  仓库和 `xcaddy` 自行构建能够获取 Buypass Go SSL 证书的 Caddy 。

## 方法一：使用 Certbot 手动获取证书（不推荐）

按照 Certbot 的官方教程进行安装，此处不再赘述。

安装完成之后，运行如下命令：

```bash
root@acme:~# certbot register -m 'YOUR_EMAIL' --agree-tos --server 'https://api.buypass.com/acme/directory'
```

此时，需要使用你的电子邮件地址在 Buypass 上注册，和 ZeroSSL 是一样的。

如果你不希望你的 web server （ Nginx Apache 都可以，但不是 Caddy ！）暂时停机的话，就选用 `webroot` 方式完成 ACME Challenge ，需要运行的命令是：

```bash
root@acme:~# certbot certonly --webroot -w /var/www/example.com/public_html/ -d example.com -d www.example.com --server 'https://api.buypass.com/acme/directory'
```

如果你用的是 Caddy ，由于 Caddy 默认的 https 重定向策略，导致 80 端口的验证几乎成为了不可能做到的事（不要说可以关掉 tls ，关掉 tls 做完验证之后就要重新开启 tls 读取获得的证书，下次续期的时候就会失败）。那就选用 `standalone` ：

```bash
root@acme:~# certonly --standalone --email 'xxx@example.org' -d 'example.org' --server 'https://api.buypass.com/acme/directory'
```

然后给 Caddy 读取证书的权限：

```bash
root@acme:~# chmod 755 /etc/letsencrypt/live/    
root@acme:~# chmod 755 /etc/letsencrypt/archive/
root@acme:~# chmod 644 /etc/letsencrypt/live/example.org/*.pem
root@acme:~# chmod 644 /etc/letsencrypt/archive/example.org/*.pem             
```

最后在 Caddyfile 配置 `tls /etc/letsencrypt/live/example.org/fullchain.pem /etc/letsencrypt/live/example.org/privkey.pem` 即可。

当然，只是做实验而已，不推荐任何人进行以上操作。同时使用 Caddy 和 Certbot 的行为纯粹是吃力不讨好，Caddy 默认会升级成 https ，并对证书进行自动续期，所以你几乎感受不到证书有效期的短暂和续期的麻烦。

## 方法二：使用 xcaddy  构建包含最新 acmez 模块的 Caddy

首先，我们需要安装最新版的 Go 。

以 Linux x86-64 为例，运行 `wget https://go.dev/dl/go1.24.1.linux-amd64.tar.gz` ，然后再运行 `rm -rf /usr/local/go && tar -C /usr/local -xzf go1.24.1.linux-amd64.tar.gz` 和 `export PATH=$PATH:/usr/local/go/bin` ， 最后运行 `go version` 验证即可。

然后，我们需要安装 xcaddy。

```bash
root@acme:~# go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
root@acme:~# export PATH=$PATH:~/go/bin
root@acme:~# xcaddy build --with github.com/mholt/acmez/v3@v3.1.1
```

最后，用 xcaddy 构建的新可执行文件替换掉原来的 Caddy 可执行文件。这一步的操作与 Caddy 的运行方式有关，此处仅展示 `systemd` 服务的替换方法。

```bash
root@acme:~# systemctl stop caddy
root@acme:~# cp /usr/bin/caddy /usr/bin/caddy.bak
root@acme:~# cp ./caddy /usr/bin/caddy
root@acme:~# chown caddy:caddy /usr/bin/caddy
root@acme:~# chmod 755 /usr/bin/caddy
root@acme:~# systemctl start caddy
```

此时再在 `Caddyfile` 中使用语法：

```nginx
{
	acme_ca https://api.buypass.com/acme/directory
	email hi@example.dev
}
```

就不会再出现 `panic` 了。

不过，如果你的网站之前已经有了 SSL 证书，那么你需要手动删除 `/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory` 下的证书文件，然后重启 Caddy 。访问你的网站，如果没有套 CF 的话，就能看到 Buypass 签发的证书了；如果你套了 CF ，那么看到的依然是边缘证书。