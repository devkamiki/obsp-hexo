---
title: Cloudflare Zero Trust Tunnel 内网穿透注意事项
date: 2025-04-02 14:10:15
tags:
  - cloudflare
  - 内网穿透
  - tunnel
  - troubleshooting
categories:
  - 日常
---
网上用 Cloudflare Tunnel 进行穿透的教程有不少，不过对连接的部分都说得比较含糊其词。

站长个人尝试穿透家里的 x86 Debian 小主机，在安装 Cloudflared 时遇到了以下问题：
- 使用 Debian 脚本，遇到 `not create ICMPv4 proxy: Group ID 0 is not between ping group 1 to 0 nor ICMPv6 proxy: socket: permission denied` 的问题。
- 使用 docker 脚本，一开始没有用宿主机网络，导致连接一直超时。
- 使用宿主机网络后，由于默认的协议是 QUIC ，会被阻断，需要改成 HTTP2 。

最终版的运行命令：
```
docker run -d --network host cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <your-token> --protocol http2
```

这样就好了……大概吧！