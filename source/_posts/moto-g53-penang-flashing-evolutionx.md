---
title: 新手技术教程：moto g53刷入Evolution X
date: 2025-05-06 17:47:44
tags:
  - flashing
  - android
  - evolutionx
  - motog53
categories:
  - 日常
---
# 前言

**解锁有风险，刷机需谨慎！学习不认真，变砖两行泪！！！**

# 找 ROM

moto 自带的系统不太好用，所以就要找个合适的 rom。不幸的是，moto g53 这个机型，代号penang，即使全球发售，在各大类原生OS（LineageOS，CalyXOS，DivestOS，crDroid等）的官网上也都找不到rom，更不要说吃上国内的Flyme和澎湃修改版了。

经过在互联网上坚持不懈地沙里淘金，我终于在 xda 找到了[一个帖子](https://xdaforums.com/t/shared-custom-rom-for-moto-g53-penang.4681392/)，楼主搜罗了一些 penang可以用的 rom，其中包括 EvolutionX，Matrixx，LineageOS 和 project2by2，只有 project2by2是来自官方的。LineageOS 的刷机包是个半成品，不能用本文所说方法刷入；Matrixx的包有问题，recovery 刷入之后会无限重启。最后我就决定先刷下载得最快（hhh）的Evolution X，再刷2by2（2by2基于LineageOS）。

Evolution X 这个系统的主要特点是“让所有手机都能获得 pixel 一样的体验”，尽可能在其他厂商的产品上还原 PixelOS 的功能。在提供各种个性化选项的同时，还提供对谷歌应用伪装成 Pixel 8 Pro 和对谷歌相册伪装成 Pixel XL（解锁无限云端备份相片）的功能。但是由于没吃过猪肉，我也不知道到底多像 Pixel，再加上我本身是一个 degoogler，深入地使用g系软件只是为了狠狠地批判它，所以我对这个系统的评价难免有失偏颇。如果你是一个重视隐私/有 FOSS 情结的人，不要刷这个 rom，买个 pixel 刷 GrapheneOS 或者 LineageOS。

![ "Heading: About Evolution X, text: Evolution X aims to provide users with a Pixel-like feel at first glance, with additional features at their disposal. Above a button: Get Android 14 for your device now. Button: Download."](https://s21.ax1x.com/2024/11/19/pAWZvng.png)
# 解 bootloader
### 前置条件

- moto 手机解 bl 的方式比较直观，不需要过小米高考或者降级参加深度测试什么的，不过解bl的前提是在开发者选项里打开 oem unlock，这个选项对于某些新设备而言，需要等待一周左右才能开启。
- 找一条可以用于 USB 调试的数据线。
- 检查你的 PC 有没有 adb 和 fastboot 这两个命令。如果没有，从[Google Android开发者](https://developer.android.com/studio)下载Android Studio Ladybug，这一步需要魔法。
- 在你放 Android Studio 的那个文件夹里找到有 adb 和 fastboot 这两个文件的目录。不管在哪里，运行的时候记得在这两个文件存在的文件夹里运行。
### 开始解锁

- 在设置里找到“关于本机”，连续点击版本号 7 次，进入开发者模式
- 进入之后，打开“ USB 调试”
- 移除所有 Google 账号
- 关机，然后用 USB 数据线将设备连接到计算机
- 运行命令 `adb -d reboot bootloader`
- 验证设备可以被计算机识别，运行 `fastboot devices`
- 进入[这个网址](https://en-us.support.motorola.com/app/standalone/bootloader/unlock-your-device-a)，拉到最下面，选择 "Proceed Anyway"
- 运行命令 `fastboot oem get_unlock_data`
- 将获得的输出（一般为五行以 (bootloader) 开头的字符串）拼接成连续的一串字符串
- 粘贴到网页中 "Paste the Device ID string into the field below to verify that your device is unlockable:" 下的空白处
- 根据指引完成确认和解锁


# 下载所需镜像

EvolutionX 的社区版 Penang 刷机包在 [GitHub](https://github.com/evox-penang/android_device_motorola_penang/releases) 可以下载，将 `boot.img`, `dtbo.img` , `evolution_penang-ota-uq1a.240105.004-01232132-COMMUNITY.zip` 和 `vendor_boot.img` 都下载下来。

# 开始刷机

- 关机状态下，长按音量下键和电源键，进入 bl
- 运行 `fastboot flash dtbo dtbo.img`
- 运行 `fastboot flash vendor_boot vendor_boot.img`
- 运行 `fastboot flash boot boot.img`
- 在设备上选择 "Recovery"
- 进入 Recovery 后，选择 "Apply Update"
- 选择 "Apply from ADB"
- 返回计算机，运行 `adb -d sideload evolution_penang-ota-uq1a.240105.004-01232132-COMMUNITY.zip`
- 侧载过程中可能会出现计算机上的进度条卡在 47% 的情况，不用着急，如果手机提示已经完成侧载，可以重启，那么就选择 "Advanced"，然后选择 "Reboot to Recovery"
- 进入 Recovery 后，选择 "Factory Reset"，然后选择 "Format data / factory reset"，确认
- 再次用上面的方法进行侧载，"Apply Update" -> "Apply from ADB" -> 运行 `adb -d sideload evolution_penang-ota-uq1a.240105.004-01232132-COMMUNITY.zip`，继续在卡 47% 后查看手机提示
- 选择 "Reboot system now"
- 如果你能够看见 EvolutionX 的 Logo，恭喜你，大功告成！
- Enjoy~