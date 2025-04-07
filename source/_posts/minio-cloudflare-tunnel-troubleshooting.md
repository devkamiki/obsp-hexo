---
title: 自建MinIO配合Cloudflare Tunnel穿透排障
tags:
  - 内网穿透
  - minio
  - cloudflare
  - troubleshooting
categories:
  - 主机折腾记录
date: 2025-04-07 16:12:40
---

大家好，今天分享一下使用家里云自建 MinIO 配合 CF Tunnel 内网穿透时遇到的问题以及解决方案。

主要的问题有三个：
- 如何使用 docker compose 的方式搭建 MinIO ？
- 第一次登录的时候，为何显示 `Invalid credentials` ？
- 为什么无法在 rclone 中使用端点域名连接，只能采用 `ip:port` 连接？

## 使用 docker compose 搭建 MinIO

对于一般的用户，我们的 `compose.yml` 应该是像这样的：

```yaml
services:
  minio:
    image: minio/minio
    command: server --console-address ":9001"  --address ":9000" /data
    environment:
      MINIO_ROOT_USER: myrootuser
      MINIO_ROOT_PASSWORD: myrootpassword
      # MINIO_SERVER_URL: "https://minio.yourdomain.com" 
      # 这个变量值的指定与否，会影响你是否能成功登录。我们先不指定
      
    volumes:
      - minio-data:/data
    ports:
      - "9000:9000"  # API 端口
      - "9001:9001"  # 控制台端口
    healthcheck:
      test: ["CMD", "curl", "-f", "https://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

volumes:
  minio-data:
```

然后运行 `docker compose up -d` 即可。国内的主机记得提前设置好速度快的 docker 镜像源。

## 内网穿透和首次登录出错

我们在 CF 准备好两个子域名，一个是 `minio.yourdomain.com` ，另一个是 `console.yourdomain.com` 。顾名思义，前者是用于 API 端点的，而后者是用于控制台的。

进入 Zero Trust 的 Network -> Tunnels，在这里你可以看到已经配置好内网穿透的家里云主机（内网穿透的教程看我之前的文章）。将 `minio.yourdomain.com` 指向 `http://localhost:9000` , `console.yourdomain.com` 指向 `http://localhost:9001`，不出意外的话，打开 `https://console.yourdomain.com` 即可成功用 `compose.yml` 中设置的 root 用户名和密码登录控制台创建存储桶了。

但是有一些朋友可能会发现，即使自己使用了完全正确的 root 用户名和密码，MinIO 控制台依然提示 `Invalid credentials` 。这是为什么呢？

就我个人的经验而言，这是因为在 `compose.yml` 中配置了 `MINIO_SERVER_URL: "https://minio.yourdomain.com"` 这一变量，而又没有为 MinIO 容器配置 SSL 证书，导致访问者到 Cloudflare 再到控制台这一段使用的是 https 协议，MinIO 内部（控制台到端点）使用的却是 http 协议。由于 MinIO 预期的是全程 https，所以连接会被拒绝。

解决的方法也很简单，首先生成一个自签名证书，假设放在 `./minio/certs` ：

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./certs/private.key -out ./certs/public.crt -subj "/CN=minio.yourdomain.com"
```

然后修改 `compose.yml`，让证书进入容器：

```yaml
services:
  minio:
    image: minio/minio
    command: server --certs-dir /certs /data
    environment:
      MINIO_ROOT_USER: myrootuser
      MINIO_ROOT_PASSWORD: myrootpassword
      MINIO_SERVER_URL: "https://minio.yourdomain.com"
    volumes:
      - minio-data:/data
      - ./minio/certs:/certs  # 挂载证书目录
    ports:
      - "9000:9000"  # API 端口
      - "9001:9001"  # 控制台端口
    healthcheck:
      test: ["CMD", "curl", "-f", "https://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

volumes:
  minio-data:
```

这样应该就没问题了。

不过，这个问题可能的成因不止一个（比如[这个](https://stackoverflow.com/questions/72291231/minio-docker-ssl-login-connection-refused)），以上只是其中之一。

## 反代端点加 rclone 无法战胜？

提要：是的，确实无法战胜！

我在使用具有公网 ip 的服务器自建 MinIO 并且用 Caddy 反代端点的时候，会出现莫名其妙的 rclone 连接问题，具体表现为：使用同样的一对 Access Key 和 Secret Key，正确进行了 rclone 配置，唯一的变量是连接的端点填写的是 `http://ip:port` 或 `https://minio.yourdomain.com` 。使用前者可以成功连接，而后者不可以。

rclone 配置如下：

```
rclone v1.69.1

- os/version: arch 25.0.0 (64 bit)

- os/kernel: 6.12.19-1-MANJARO (x86_64)

- os/type: linux

- os/arch: amd64

- go/version: go1.24.0

- go/linking: static

- go/tags: none
```

`.config/rclone/rclone.conf`：

```sh
[minio]                                                                                                   
type = s3                                                                                                    

provider = Minio                                                                                              

access_key_id = xxx                                                                    

secret_access_key = xxxx                                                  

region = us-east-1                                                                                            

endpoint = https://minio.example.dev                                                                              

disable_http2 = true                                                                                          

force_path_style = true                                                                                       

v4_auth = true                                                                                                

no_check_certificate = true                                                                                                   

no_system_metadata = true
```

“无法连接” 在我这里的表现是这样的：

```bash
[yuki@manjaro ~]$ rclone ls minio: -vv

2025/04/02 22:11:34 DEBUG : rclone: Version "v1.69.1" starting with parameters ["rclone" "ls" "minio:" "-vv"]
2025/04/02 22:11:34 DEBUG : Creating backend with remote "minio:"
2025/04/02 22:11:34 DEBUG : Using config file from "/home/yuki/.config/rclone/rclone.conf"
2025/04/02 22:11:36 DEBUG : 5 go routines active
2025/04/02 22:11:36 NOTICE: Failed to ls: operation error S3: ListBuckets, https response error StatusCode: 403, RequestID: 183285C7252CB364, HostID: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8, api error SignatureDoesNotMatch: The request signature we calculated does not match the signature you provided. Check your key and signing method.
```

起初，我看了一个类似的 GitHub Issue（对不起时间有点久找不到了，找到补上），是使用 Nginx 进行反代的，然后通过加特定的 headers 解决了问题。所以我想肯定是我的 Caddy 功夫还不过关，写的反代有问题导致的。但是在使用 CF 穿透的时候又一次遇到了这个问题，而这一次没有任何 web server 安装在家里云小主机上。

我对此百思不得其解，在电脑前折腾了整整一天一夜，终于……放弃了……

当我坐在马桶前沉思的时候，突然灵光乍现：有没有可能，这不是 MinIO 的问题，不是 Caddy 的问题，不是 CF 的问题，而是 rclone 的问题？

于是我立马使用 MinIO 官方的命令行工具 `mc` 使用一模一样的 Access Key 和 Secret Key 对，并且用 `https://域名` 作为端点。令我吃惊的是，这一次我可以成功进行 `ls` 操作了：

```bash
[yuki@manjaro ~]$ mc ls minio    
[2025-04-02 15:42:43 CST]     0B test/      
[yuki@manjaro ~]$ mc ls minio/test                  
[2025-04-02 16:40:17 CST]  33KiB STANDARD <picturename>.jpg
```

完全正确，没有出现 403 错误。

看来，我的 https 设置并没有问题，问题的确是出在 rclone 上。

在 Deepseek 和 Claude 的帮助下，我对比了两个工具的请求头（完整输出太长，只截取重要部分），已经把 troubleshooting 步骤[发在 rclone forum](https://forum.rclone.org/t/signature-doesnt-match-for-minio-behind-cloudflare-tunnel/50691)：

运行 `mc --debug ls one/test`  得到的 headers：

```bash
X-Amz-Date: 20250402T142607Z  
Authorization: AWS4-HMAC-SHA256 Credential=<accesskey>/20250402/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=xxx  
```

运行 `rclone -vv --dump headers --dump auth ls minio:`  得到的：

```bash
X-Amz-Date: 20250402T142710Z  
Authorization: AWS4-HMAC-SHA256 Credential=<accesskey>/20250402/us-east-1/s3/aws4_request, SignedHeaders=accept-encoding;amz-sdk-invocation-id;amz-sdk-request;host;x-amz-content-sha256;x-amz-date, Signature=xxx  
```

可以看到 `Authorization` 稍有不同，这是因为 rclone 默认为所有 S3 兼容存储添加了以下 headers，而这并不是 MinIO 期望的: `accept-encoding;amz-sdk-invocation-id;amz-sdk-request;`。

[类似的问题发生在 Exoscale 对象存储上](https://forum.rclone.org/t/signaturedoesnotmatch/13428/3)。

> Rclone uses the official AWS SDK so I'd guess that this is a bug in exoscale.

问题的原因找到了，解决的方法嘛，那就是……还没有。rclone 暂时没有提供在配置文件里修改请求头的方法（也可能有，但是我没有成功），所以只能先用其它工具了。