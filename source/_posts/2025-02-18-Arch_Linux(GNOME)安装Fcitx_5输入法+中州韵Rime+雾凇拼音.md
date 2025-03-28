---
layout: post
cid: 18
title: Arch Linux(GNOME)安装Fcitx 5输入法+中州韵Rime+雾凇拼音
slug: arch-linux-gnome-fcitx5-rime-ice
date: 2025/02/18 09:25:00
updated: 2025/02/18 09:31:23
status: publish
author: 神木友希
categories: 
  - 主机折腾记录
tags: 
  - rime
  - fcitx5
  - arch linux
  - gnome
  - rime-ice-git
desc: 
img: 
---


以下配置适用于GNOME桌面，KDE和Xfce用户请参考其它文章。

## 安装Fcitx 5和中州韵Rime：

Fcitx 5用AUR helper装：

`yay -S fcitx5 fcitx5-rime fcitx5-configtool fcitx5-gtk fcitx5-qt fcitx5-config-qt`

我一直用sudo yay，发现以sudo运行的时候会提示不建议，直接yay即可。

`nano`一下`/etc/environment`，加入如下内容

```
XMODIFIERS=@im=@fcitx
QT_IM_MODULE=fcitx
GTK_IM_MODULE=fcitx
```

## 安装必要的GNOME组件：

[安装Input Method Panel](https://extensions.gnome.org/extension/261/kimpanel/)

重启一下，打开Fcitx 5配置，把中州韵Rime添加到键盘里面。

## 安装雾凇拼音：

`yay -S rime-ice-git`

改一下Rime的配置：`nano /.local/share/fcitx5/rime/default.custom.yaml`，写入：

```
patch:
  # 仅使用「雾凇拼音」的默认配置，配置此行即可
  __include: rime_ice_suggestion:/
  # 以下根据自己所需自行定义，仅做参考。
  # 针对对应处方的定制条目，请使用 <recipe>.custom.yaml 中配置，例如 rime_ice.custom.yaml
  __patch:
    key_binder/bindings/+:
      # 开启逗号句号翻页
      - { when: paging, accept: comma, send: Page_Up }
      - { when: has_menu, accept: period, send: Page_Down }
```


参考文献：https://github.com/iDvel/rime-ice