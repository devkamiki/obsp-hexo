---
title: 在局域网设备上搭建 OpenCloud.eu 实例
date: 2025-06-26 15:00:15
tags:
  - opencloud
  - traefik
  - docker
categories:
  - 主机折腾记录
---
## 前言

[OpenCloud](https://opencloud.eu) 是自建云存储解决方案中的新秀，类似于 NextCloud 和 ownCloud，但更加轻量化、现代化，强调安全、简洁、数字主权与团队合作，由 mailbox.org 的母公司 Heinlein Support 旗下团队研发。

由于 OpenCloud 是 2025 年 1 月新推出的产品，因此应用市场中可选的插件暂时很少。

今天我们介绍如何通过 docker compose 在局域网设备上搭建一个 OpenCloud 实例，并通过 Traefik 反代实现局域网域名访问。

在进行安装前，请确保 OpenCloud 将会运行的机器上安装着任意 Linux 发行版或 MacOS 或 WSL，已经安装 git 和 docker compose，并且至少有 1 GHz 单核 CPU 和 512 MB 内存。具体的配置可以参考[官方文档](https://docs.opencloud.eu/docs/admin/getting-started/requirements)。

## 安装步骤

OpenCloud 大大简化了局域网中 Collabora Office 实例与云盘连接的配置。由于 Collabora Office 需要 https 连接，那么你需要自行进行局域网域名的自签名 SSL 证书设置。当然，你也可以关闭强制 https，但 Collabora Office 依然强制要求每一段连接中的协议一致，也就是说，必须全程使用 http 或 https。 如果你曾经在局域网中安装 NextCloud 和 Collabora Office 并且只使用 `<ip>:<port>` 连接，那么你就会发现，当使用 Cloudflare Tunnel 的时候，由于浏览器客户端到 CF 边缘服务器是 https，而容器内部是 http，你将无法编辑 NextCloud 中的文档。OpenCloud 的懒人版配置文件自动为 Collabora 和 OpenCloud 分配了局域网域名并且生成了证书，你只需在访问的客户端中为这些域名指定 ip 地址并手动信任证书即可。

在安装之前，确保你的机器上没有运行 Apache, Nginx 或 Caddy 等网络服务器监听 80 和 443 端口。然而，这并不意味着你需要停止正在运行的网络服务器——例如，如果你使用 Caddy 和 Tailscale 进行局域网域名访问，并且希望继续将 OpenCloud 实例通过现有的网络服务器反代，那么你需要注释掉配置文档中关于 Traefik 的部分，并修改默认配置中的域名，最后需要生成自签名证书。由于我没有进行尝试，无法提供准确的配置文件参考，未来可能会补上。

克隆项目仓库（国内的机器请自备镜像）：

```bash
git clone https://github.com/opencloud-eu/opencloud.git
```

进入 compose 目录：

```bash
cd opencloud/deployments/examples/opencloud_full
```

Docker，启动！

```bash
docker compose up -d
```

接下来我们需要在客户端机器的 `/etc/hosts` 文件中添加：

```plaintext
192.168.128.28        cloud.opencloud.test
# ip 地址改成 OpenCloud 宿主机的局域网 ip
192.168.128.28        collabora.opencloud.test
# 如果是本机就改成 127.0.0.1
192.168.128.28        wopiserver.opencloud.test
# 读到这里你是不是想问
# 那手机 (Android, iOS) 怎么办
# 呃……我也不知道
```

在浏览器访问 https://collabora.opencloud.test, 会提示有安全风险，因为证书是自签名的。选择接受风险并继续即可。如果能看到 OK 字样，说明 Collabora Office 服务器正常运行。

此时可以访问 https://cloud.opencloud.test, 接受证书后使用用户名和密码均为 admin 登录，即可享受 OpenCloud 啦！

Btw，插件市场没有日历和任务应用，我去看了下官方文档，OpenCloud 的 CalDAV 和 CardDAV 功能是通过 Radicale 实现的，在配置文档中取消注释这一部分即可。如果你看过 Observer's Space 之前的详细版 [Radicale 设置教程](https://obsp.de/p/2025-03-05-caldav%E5%8F%8Acarddav%E6%9C%8D%E5%8A%A1%E5%99%A8radicale_v3%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97+caddy_v2%E4%B8%80%E8%A1%8C%E5%8F%8D%E4%BB%A3/), 就不需要依赖自动配置了，可以自己动手丰衣足食！

OpenCloud 的文件浏览界面如图，我挺喜欢这个 UI 的：
![OpenCloud Interface](https://i.111666.best/image/PG3DnY9ql8nqgk421Xz1sc.png)