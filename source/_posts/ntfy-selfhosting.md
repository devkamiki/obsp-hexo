---
title: docker 自建 Ntfy 实例以及提醒功能（接入 Uptime Kuma，Wallos 等）实战
tags:
  - ntfy
  - docker
  - OSS
categories:
  - 主机折腾记录
date: 2025-04-10 12:27:54
---

# 前言

大家好，两个星期没更新了，~~想必根本没有人想我罢~~

今天我们来搭建一个 Ntfy 实例，照例还是使用非常方便的 docker compose。

Ntfy 的工作原理非常简单，以官方实例 `ntfy.sh` 为例，这个实例是没有密码保护的，当用户在订阅任意的主题（通常是 `https://ntfy.sh/themename` 的格式）时，任何发送到这个主题的通知信息都会被用户获取。因此，一些使用 Ntfy 官方实例进行通知推送的软件（如 Fluffychat）会自动生成一串随机字符，防止其他用户不慎和你订阅了同一个主题而获取你的通知信息。然而，使用诸如官方实例或是 [tchncs.de 的 ntfy 实例](https://push.tchncs.de)这样的实例终究会因为主题的开放而存在一定的隐私或安全风险，更多的小伙伴还是希望能够用密码保护通知内容。那么今天我们就来一起看看如何搭建和管理自己的 Ntfy 实例吧！

# 部署过程

Ntfy 支持多种部署方式，并且有详尽的[官方文档](https://docs.ntfy.sh/install/)，感兴趣的读者可以自行查阅。

直接上 `compose.yml` ：

```YAML
services:                                                                            

  ntfy:                                                                              

    image: binwiederhier/ntfy                                                        

    restart: unless-stopped                                                          

    container_name: ntfy                                                             

    environment:                                                                     

      NTFY_BASE_URL: https://ntfy.example.org                                          

      NTFY_CACHE_FILE: /var/lib/ntfy/cache.db                                        

      NTFY_AUTH_FILE: /var/lib/ntfy/auth.db                                          

      NTFY_AUTH_DEFAULT_ACCESS: deny-all                                             

      NTFY_BEHIND_PROXY: true                                                        

      NTFY_ATTACHMENT_CACHE_DIR: /var/lib/ntfy/attachments                           

      NTFY_ENABLE_LOGIN: true

    volumes:                                                                     
     - ./:/var/lib/ntfy                                                                                                                

    ports:                                                                       
     - 127.0.0.1:8080:80                                                           

    command: serve
```

这样子，一个具有访问控制功能的 Ntfy 实例就建成了。

请配置反向代理（Nginx, Caddy, Apache 等），将 `https://ntfy.example.org` 反代到 `localhost:8080`。如果一切顺利，访问 `https://ntfy.example.org` 即可看到 Ntfy 网页客户端的界面！

此时，默认对所有主题的控制策略都是 `deny` ，我们需要创建用户，并且为它们分配可以读或写某些主题的权限。

当没有进行容器化安装时，用户的创建和管理都是通过 `ntfy` 命令来完成，而可执行的 `ntfy` 位于 docker 容器内部，此时需要指明 `ntfy` 所在的容器名称。由于我们在 `compose.yml` 中进行了命名：`container_name: ntfy` ，故该容器的名称应为 `ntfy` 。

创建用户的命令如下：

```bash
# 创建管理员，可以访问所有的主题。adminname改成你想要的管理员用户名
docker exec -it ntfy ntfy user add --role=admin adminname

# 创建普通用户，默认无法访问任何主题。username改成你想要的用户名
docker exec -it ntfy ntfy user add --role=user username

# 修改用户的访问权限，格式如下
docker exec -it ntfy ntfy access <username> <theme> <read-write/read-only/write-only/deny>

```

比如说，Alice 希望自己能够获取所有主题的通知，那么就执行 `docker exec -it ntfy ntfy user add --role=admin alice` ，根据提示输入密码。

Alice 运行着一个 Uptime Kuma 实例，她希望在服务不可用的时候收到 Uptime Kuma 向主题 `uptime` 的推送通知，但是又不希望 Uptime Kuma 获取到其他主题的推送通知，于是她为此创建了一个普通用户：`docker exec -it ntfy ntfy user add --role=user uptimekuma` , 根据提示输入密码，然后给予对应的权限： `docker exec -it ntfy ntfy access uptimekuma uptime read-write` 。完成之后，在 Uptime Kuma 中填写用户名 `uptimekuma` 、密码、实例网址和主题名称 `https://ntfy.example.org/uptime` 即可。

# 实用案例

能够使用 Ntfy 进行推送通知的应用还有很多，如哪吒探针和 Wallos 等。我们也可以通过编写自动化脚本，在任务触发或状态改变的时候订阅了某个主题的客户端发送消息。

例如，前段时间我让 Claude 写了个脚本，当有进程对 CPU 的占用超过10%的时候，就把进程的名称以及占用情况输出到一个文件中，并且将进程停止，报告到我的 ntfy 实例的一个特定主题。

脚本内容如下：

```bash
#!/bin/bash

# 设置变量
LOG_FILE="cpu_limit.log"
NTFY_URL="https://ntfy.example.io/xxxxxx"
NTFY_USER="xxx"
NTFY_PASS="xxx"
CPU_THRESHOLD=10

# 创建日志文件或清空现有文件
> $LOG_FILE

echo "$(date): 开始监控 CPU 使用率..." >> $LOG_FILE

# 持续监控进程
while true; do
    # 获取 CPU 使用率超过阈值的进程列表
    processes=$(ps aux | awk -v threshold=$CPU_THRESHOLD '$3 > threshold {print $2, $3, $11}')
    
    if [ -n "$processes" ]; then
        echo "$(date): 发现 CPU 使用率超过 ${CPU_THRESHOLD}% 的进程:" >> $LOG_FILE
        
        # 遍历每个超过阈值的进程
        echo "$processes" | while read pid cpu_usage process_name; do
            echo "  PID: $pid, 名称: $process_name, CPU使用率: ${cpu_usage}%" >> $LOG_FILE
            
            # 停止进程
            kill $pid
            kill_status=$?
            
            if [ $kill_status -eq 0 ]; then
                echo "  已停止进程 $pid ($process_name)" >> $LOG_FILE
                
                # 发送通知到 ntfy
                curl -u "$NTFY_USER:$NTFY_PASS" -H "Title: CPU 限制触发" \
                     -d "进程 $process_name (PID: $pid) 的 CPU 使用率达到 ${cpu_usage}%，已被停止。" \
                     $NTFY_URL
            else
                echo "  无法停止进程 $pid ($process_name)" >> $LOG_FILE
            fi
        done
    fi
    
    # 暂停 5 秒后再次检查
    sleep 5
done
```

亲测有效~