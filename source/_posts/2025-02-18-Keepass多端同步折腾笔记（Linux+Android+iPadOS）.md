---
layout: post
cid: 19
title: Keepass多端同步折腾笔记（Linux+Android+iPadOS）
slug: keepass-sync
date: 2025/02/18 09:53:00
updated: 2025/02/18 10:21:48
status: publish
author: 神木友希
categories: 
  - 日常
tags: 
  - rclone
  - keepass
  - nextcloud
desc: 
img: 
---


- 踩坑：用Argon2加密的iOS用户，一定要记得去加密方式里面把内存调到32MB以下，默认好像是64；如果保持默认的话由于iOS对每个应用的内存限制，在使用输入法内嵌自动填充的时候会死翘翘，崩溃。
- 同步方案：
- 密码在Nextcloud上存一份，手机本地存一份，PC本地存一份。
- 安卓手机本地文件夹用[Foldersync](https://foldersync.io/)和Nextcloud进行双向同步（想要用free and open source software的话，下载[rclone](https://rclone.org/)的安卓GUI [Round Sync](https://roundsync.com/)或者[Termux](https://termux.dev/en/)命令行模拟器直接用[rclone](https://rclone.org/)吧，我只用过前者，不支持双向同步功能，作者说这个功能实现遇到了困难，搁置了。rclone的本体是支持[bisync](https://rclone.org/bisync/)的，但是我懒得在小屏幕上折腾命令行），设置定时任务，每小时一次。我用的安卓Keepass客户端是[KeepassDX](https://www.keepassdx.com/)。
- Linux桌面将Nextcloud文件夹挂载到本地文件目录，双向同步对文件的修改；或者用rclone连接远端，`crontab -e`定时运行`rclone bisync`。客户端+浏览器扩展是[KeepassXC](https://keepassxc.org/)。（我用的浏览器是Brave，需要注意KeepassXC不支持Flatpak版的Brave，请用官方脚本/软件源安装）
- iPad用[Strongbox](https://strongboxsafe.com/)，fetch远程的WebDAV文件夹，或者[奇密（Fantasy Pass）](https://github.com/kaich/FantasyPass)，后者很便宜，可惜是闭源的，而且看起来好像没有维护了。Strongbox免费版满足大部分需求，并且ui、自动填充2FA和数据库警报都很好用，生成密码的功能也很全面（还可以加盐，但是我不太懂为什么加盐要在密码管理器生成明文密码这一步进行）。
- 这套方案唯一的问题就是Nextcloud对数据的处理有时候不太尽如人意，经常因为文件名相同内容不同就报错，但是直接忽略，取最新版就OK了。或者换别的WebDAV/S3提供商，例如[Koofr](https://koofr.eu)，[Hetzner Storage Box](https://www.hetzner.com/storage/storage-box/)，[Scaleway 对象存储](https://www.scaleway.com/en/object-storage/)，[Storj](https://www.storj.io/)等等。（之前同步的时候用过[Backblaze B2](https://backblaze.com)，我的感想是最好不要用B2，B2的list object操作是计费的，rclone非常吃这个操作，尤其是文件版本多/小文件多的情况，即使在同步命令里加上`--fast-list --local-no-check-updated`也能把一天中的2500次操作吃光。）