---
layout: post
cid: 4
title: tabby使用插件cloud sync settings利用gitlab gist同步设置遇到的问题
slug: tabby-cloud-sync-settings-gitlab-gist
date: 2025/02/09 16:49:00
updated: 2025/03/05 12:03:54
status: publish
author: 神木友希
categories: 
  - 主机折腾记录
tags: 
  - troubleshooting
desc: 
img: 
---


我的PC系统是Manjaro Linux，用tabby当ssh工具。
tabby的官方同步需要自己host一个实例，我不太信得过自己的服务器手艺~~99.9%downtime~~，又想要经常备份，巧的是tabby插件市场有一个叫cloud sync settings的插件，可以通过s3/webdav/gist同步设置，于是我就选了gitlab的snippet（gist同步插件上的提示是github或者gitlab，但是据说gitee也行。没试过）
本来用得好好的，每20s同步一次，偶尔会报错，但是大多是网络问题，过一会就自己恢复了。
今天在使用的时候突然持续报错，每次同步都出错，我一看logs：
```
error: Upload settings file from local| \
Exception: Error: Request failed with status code 400
```
我心想：莫非是access token过期了？
然而登上gitlab一看，我设置的期限是1年，还有11个月才会过期。
我试着rotate了一下，换了新的token，报错继续。
此时我发现test connection的时候是无异常的，报错是在开始尝试上传本地配置的时候出现的。
我试着把gist id留空，让它创建新的snippet，这下没问题了。
过了几分钟，我打开gitlab，看到snippet的编辑时间是just now，说明把配置同步到这个snippet没有问题，只有那个旧的snippet是不能用的。
我把gist id改回旧snippet的，果然还是不行。
这下只好用这个新的snippet了。这个插件的同步都是单向同步，可以选择是云端到本地还是本地到云端，我直接把最新的设置从本地传上去，继续使用。
（不过除了刚configure完那一下的方向可以选择，以后的同步都是本地到云端，这个不能改。也就是说假如计算机A、B都连接了同一个云端文件，A做出的改动传到云端，此时你打开计算机B，是不会看到A做出的改动的。B本地的配置文件会直接覆写云端文件。）
虽然找到了问题所在，但是我不知道原因是什么，等我去提个issue看看吧……

收到的回复：
niceit
niceit
 left a comment 
(niceit/tabby-cloud-sync-settings#64)

@f-engg I am not sure yet about the Gitlab issue with snippet, but as I know that Github limited the requests, I think Gitlab is also did the same thing, you can try to switch to other reliable provider like Dropbox, or S3.