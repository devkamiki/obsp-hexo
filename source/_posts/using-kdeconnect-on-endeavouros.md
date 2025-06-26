---
title: 在 EndeavourOS 上使用 KDE Connect
date: 2025-06-20 12:22:38
tags:
  - android
  - endeavouros
  - kde
categories:
  - 日常
---
如果你使用默认设置安装 EndeavourOS，那么在使用 KDE Connect 时就会发现无法寻找到局域网中的其它设备。

不必担心，这是默认的防火墙设置导致的。

右键点击任务栏的防火墙图标，选择“编辑防火墙设置”，如图所示。

![Select "Edit Firewall Settings"](https://i.111666.best/image/xVhNMQ7AqDwe1hgRrfaepE.png)

在当前网络使用的策略组中勾选 "KDE Connect".

![Allowing KDE Connect in Group Public or whichever you are using](https://i.111666.best/image/dKhpt8aSfwJdst1T7IXMz6.png)

返回 KDE Connect 设置，刷新设备列表，此时你应该能够看到局域网中的设备。