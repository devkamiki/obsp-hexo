---
layout: post
cid: 23
title: SearXNG接入Open WebUI显示Error searching的解决方法
slug: searxng-openwebui-error-searching
date: 2025/03/03 10:22:00
updated: 2025/03/03 10:24:46
status: publish
author: 神木友希
categories: 
  - 主机折腾记录
tags: 
desc: 
img: 
---


可能是由于没开启JSON输出导致的，

在您的`/usr/local/searxng-docker/searxng/settings.yml`加入如下内容：

```
search:
  formats:
  - html
  - json
```

OpenWebUI的搜索字符串不是%s，也不是空白，是`<query>`，例如使用本站SearXNG实例进行搜索，要在联网搜索处配置的URL是`https://q.obsp.de/search?q=<query>`。