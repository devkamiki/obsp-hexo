---
title: EndeavourOS 日用初探以及 Nvidia GPU 驱动安装
tags:
  - endeavouros
  - troubleshooting
  - arch
  - kde
  - fcitx5
categories:
  - 日常
date: 2025-04-21 16:22:03
references:
  - '[PRIME](https://wiki.archlinuxcn.org/wiki/PRIME#%E9%85%8D%E7%BD%AE%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E4%BD%BF%E7%94%A8_GPU_%E6%B8%B2%E6%9F%93)'
  - '[ArchLinux 下 AMD 集显 NVIDIA 独显双显卡方案](https://www.hhhil.com/posts/amd-nvidia-gpu/)'
---

# 前言

一个月前在我的不懈折腾下终于装上了 N 卡驱动开始快乐工作的 Manjaro，还没来得及承担打游戏和跑 LLM 的光荣任务就莫名其妙地壮烈牺牲了，具体体现在系统内选择关机之后有时无法正常进入关机状态，会冒出一堆内含关键词 nvidia 的报错并陷入循环，且风扇开始疯狂地呼呼作响。时间一长，渐渐地连开机都出了问题，经常会启动失败。于是一个念头逐渐在我脑海中成型：要不重装一下吧……

Manjaro 作为 Arch 衍生物中的日用之王，带来的负面体验很少， N 卡的问题算是其中之一，不过这不是 Manjaro 的问题。

热爱折腾的我，这次将目光转向了好评如潮的新秀 EndeavourOS。

# 缺点

先说缺点：
- EndeavourOS 对于有魔法需求的国内用户不是很友好，当使用 Clash Verge Rev 之类的工具时，大部分的应用都会绕过代理，只能通过开 TUN 模式解决。
- 遇到一些不知道是（EndeavourOS 的问题）还是（KDE 的问题）还是（Wayland 的问题）的问题，
	- 某网盘软件 AppImage 版本上传和下载的按钮没有反应，而 Manjaro + GNOME + X11 不会，由于无法控制变量，此处暂时没有定论
	- Minecraft Launcher 每次开机后只能成功启动一次，此后再启动就会提示错误，必须重启计算机，我没有在任何其他发行版上用过 Minecraft Launcher，此处暂时没有定论
	- kmail 会开启后闪退
	- KDE Connect 一直找不到我的移动设备，排查确定两台设备接入了同一局域网而且能互相通信，不太清楚问题所在
	- CUPS 不能像在 Manjaro 那样自动发现局域网的打印机，不过输入 `socket://hostname:9100` 就好了

# 优点，N 卡用户请看

剩下的都是优点：
- 如果你用的是新的 Nvidia 显卡，那么 EndeavourOS 绝对是 Arch 衍生物中不得不品的一环
	- EndeavourOS ISO 镜像的 boot 引导中可以选择普通还是 N 卡设备，一般来说在这一步选择 N 卡设备，boot 进去后 install，后续就不需要再考虑驱动的问题了
	- 但是我在这一步没能 boot 成功，报错的内容忘记记下来了（……）最后选择的是普通版
	- 如果是这样的话也没关系，EndeavourOS 的开发者们维护的 AUR 仓库 `endeavouros` 中有一个非常便捷的命令——`nvidia-inst`，功能和使用指南可以参考[官方文章](https://discovery.endeavouros.com/nvidia/new-nvidia-driver-installer-nvidia-inst/2022/03/)和[社区问答](https://forum.endeavouros.com/t/what-exactly-does-nvidia-inst-do/54997/2)，使用它安装 Nvidia 驱动仅需 2 步：
		- `yay -S nvidia-inst`
		- `nvidia-inst`
	- 如果你使用的是其他 Arch 衍生物发行版，只需按照上文中的社区问答添加 endeavouros 仓库和镜像即可
	- 如果你用的是不再受 Nvidia 官方 Arch 仓库驱动支持的旧版显卡，在命令中加入 `--nouveau` 或 `-n`
	- 如果你需要 Bumbleweed，在命令中加入 `-b`
- EndeavourOS 自带镜像选择器 `reflector` 的 GUI，名叫 Reflector Simple，直接勾选国家或地区即可将该地区的推荐镜像源添加到镜像列表，也可以在 GUI 上选择按照速率/延迟等指标排序镜像
- 开发者的审美不错，
	- 默认的黑暗紫色主题 KDE 和 Xfce 桌面都很漂亮
	- 预装的应用不多也不少，不像 Manjaro 那样大而全（自带相当于 pacman+AUR+flatpak GUI 的应用商店这种~~邪恶的~~东西 www），也不像原味 Arch 一样只有骨架，可以说是简约主义和日用功能的完美平衡

# 后记

如果你和我一样，对技术一窍不通，只因 AUR 的魔力使用 Arch 的话，或者你希望在 Linux 上充分发挥 Nvidia GPU 的性能的话，EndeavourOS 非常值得一试。

最后还是要说一些和 EndeavourOS 或者我的 PC 硬件没有太大关系的碎碎念，本应另开一文，但是以下经验也算是日用体验的一部分吧！

## Rime

Rime 在 KDE + X11 下的配置应该是这样的，在设置-虚拟键盘中选择 fcitx5，输入法中选择中州韵，在 `/etc/environment` 键入：

```shell
XMODIFIERS=@im=fcitx  
INPUT_METHOD=fcitx  
SDL_IM_MODULE=fcitx  
GLFW_IM_MODULE=fcitx  
QT_IM_MODULE=fcitx  
GTK_IM_MODULE=fcitx
```

那这个时候就有小伙伴发现，如果使用的是 Wayland，fcitx 5 启动时会提示你删除 `QT_IM_MODULE` 和 `GTK_IM_MODULE` 这两行。

删除之后没有问题的话可以直接跳过这一部分，不过我在日用的时候发现，如果去掉这两行，时不时会产生丢字母的问题。具体表现为尝试输入拼音 `ceshi` ，只有 `cshi` 被保留在输入法候选框中，而 `e` 作为单独的字符掉落下来，键入到文本框中。这个问题说大不大说小不小，打一个完整的句子时非常烦人。即使在虚拟键盘下启用 fcitx 5 Wayland 启动器，也解决不了这个问题。

有一天我灵机一动，加上了 `QT_IM_MODULE` 和 `GTK_IM_MODULE` 这两行，然后这个问题就再也没有出现过……

## OnlyOffice

这个问题是 N 卡和窗口系统的相性导致的。

问题描述：在使用 X11 窗口系统的时候，安装 N 卡驱动之后，OnlyOffice 的显示比例会变得特别大，参照[这个 Issue](https://github.com/ONLYOFFICE/DesktopEditors/issues/324)。有时候随着 OnlyOffice 的版本更新，这个问题会自行消失，不过我最近还是遇到了。

换用 Wayland 就好了。


## 2025-6-9 更新：双显卡配置，Steam 游戏启动选项

先贴我的系统信息，一个英伟达独显，一个 AMD Radeon 集显：

```
Operating System: EndeavourOS 
KDE Plasma Version: 6.3.5
KDE Frameworks Version: 6.14.0
Qt Version: 6.9.1
Kernel Version: 6.14.10-arch1-1 (64-bit)
Graphics Platform: Wayland
Processors: 16 × AMD Ryzen 7 7735H with Radeon Graphics
Memory: 38.4 GiB of RAM
Graphics Processor 1: NVIDIA GeForce RTX 4060 Laptop GPU
Graphics Processor 2: AMD Radeon 680M
Manufacturer: ASUSTeK COMPUTER INC.
Product Name: ASUS TUF Gaming A15 FA507NV_FA507NV
System Version: 1.0
```

我在运行 `nvidia-inst` 后又运行了 `paru -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon  lib32-vulkan-radeon libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau` ，然后就可以愉快食用了（如果不装集显的驱动就会两个卡槽都用 N 卡的 OpenGL，显示在系统信息里的会是两个英伟达 GPU）。

为什么突然现在更新呢，因为我把集显的事彻底忘在了脑后，我天真地以为只要假装集显不存在就好了。直到用 Steam 和 Proton 启动 2077 的时候一直报错（如图所示），但是正确安装驱动的情况下应该是不会有这个 OpenGL 的问题的，再加上 ProtonDB  上评级为 Native 和 Platinum 的红弦俱乐部和逆转裁判123在启动时黑屏，我不由得开始怀疑是不是自己的显卡驱动真的有问题。

![cyberpunk2077报错](https://i.111666.best/image/yHlbHLCCrPaB3Jv5yTi1BI.png)

在这个报错之前我完全没怀疑是显卡的问题，试了很多种启动参数都没能玩上游戏，但是这个关键的错误启发了我：2077 可能在尝试使用我的集成显卡……

于是，我在启动参数中强制游戏使用独显：`_NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia %command%`，问题迎刃而解。

最后我就去把集显驱动装上了……就是这样……
