---
layout: post
cid: 26
title: Rclone以WebDAV方式挂载Hetzner Storage Box
slug: rclone-mount-hetzner-storage-box
date: 2025/03/10 12:12:00
updated: 2025/03/10 12:41:59
status: publish
author: 神木友希
categories: 
  - 日常
tags: 
  - rclone
  - webdav
  - hetzner
  - storage
desc: 
img: 
---


- 疑似有点太水了，想不出这个有什么出教程的必要（不是）。
- 本来想尝试[Rclone UI](https://rcloneui.com/)，奈何AppImage版在Arch上根本启动不了，我试着在AUR里找了一下`yay rclone-ui`，居然没找到……原来AUR也不是万能的，那就继续用命令行的原版Rclone吧。
- Hetzner买Storage Box的过程就跳过了。因为hz是PAYG的模式，所以账单一般下个月才会出。
- 进控制台：https://robot.hetzner.com/storage，Reset一下根用户密码。然后打开WebDAV、SSH，用WebDAV或SSH工具进去创建一个空的子文件夹，如果没有子文件夹的话是无法创建子账户的。
- 然后去sub-account创建一个子账户，打开WebDAV和External-access，以便在互联网上访问。
- 子账户的WebDAV地址是https://uxxxx(换成你的主账户id)-subx(换成你的子账户id).your-storagebox.de，账户是uxxxx-subx，密码是你设置的密码。
- Rclone官方一键安装脚本：`sudo -v ; curl https://rclone.org/install.sh | sudo bash`
- 按照提示操作选择新建远端，起个名字（以Hetzner为例）选中WebDAV，然后把上面的地址、用户名和密码填进去，剩余部分留空即可。
- 挂载：`rclone mount hetzner: /path/to/mnt `
- 这个时候就会发现Rclone提示这个远端不支持流式传输（streaming），需要加上关于缓存文件的参数。
- 那么命令就可以修改成`rclone mount hetzner: /path/to/mnt --vfs-cache-mode writes`
- 运行命令的时候会发现只能前台运行，如果退出执行命令就会导致停止挂载。
- 解决办法也很简单，加入后台运行参数即可：`rclone mount remote:path /path/to/mount --vfs-cache-mode writes --daemon`。