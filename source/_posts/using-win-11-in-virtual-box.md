---
title: 在 Virtual Box 中安装 Windows 11 虚拟机
date: 2025-05-15 15:55:20
tags:
  - windows
  - virtualbox
categories:
  - 日常
---
大家好，今天分享一下在 Arch Linux (EndeavourOS) 上使用 Virtual Box 启动 Windows 11 家庭版虚拟机的过程。

# 下载 iso

我们可以在[这里](https://www.microsoft.com/en-us/software-download/windows11)的页面下拉，找到 "Download Windows 11 Disk Image (ISO) for x64 devices", 按照指引操作即可。

# 配置虚拟机

- 打开 Virtual Box，选择 “新建”
- 输入虚拟机名称
- 在“虚拟光盘”处选择之前下载的 ISO 文件
- “类型”选择“Microsoft Windows”
- “版本”选择“Windows 11（64-bit）”
- 勾选“跳过自动安装”
	- 接下来我们需要进行一些手动操作
- 分配CPU、RAM和硬盘，满足[这里的要求](https://aka.ms/WindowsSysReq)即可
- 确认创建虚拟机
- 在虚拟机的“设置-主板”中，取消勾选 EFI 和 Secure Boot
	- 这是因为当启用 Secure Boot 和 EFI 的时候，无法从光驱启动，会造成启动失败
	- 但是 Windows 11 安装程序的系统监测程序会检查 UEFI 和 Secure Boot 的可用性，如果无法通过检查，安装程序就会终止
	- 我们稍后再来解决这个问题
- 在虚拟机的“设置-处理器”中，勾选“启用PAE/NX”
- 启动！

# 开始安装

- 选择键盘配置和地区
- 勾选“在这台电脑上安装 Windows 11”
- 此时不要选择版本或输入产品密钥
- 使用虚拟机软键盘键入“Shift+F10”
- 执行命令 `regedit` 打开注册表
- 导航到目录：HKEY_LOCAL_MACHINE -> SYSTEM -> Setup
- 在 Setup 下新建一项，命名为 LabConfig
- 添加四个 DWORD 值，名称分别为 `BypassTPMCheck`， `BypassRAMCheck` ， `BypassCPUCheck` 和 `BypassSecureBootCheck`，均赋值为1，用来绕过监测程序对 TPM 2.0，处理器、内存和 Secure Boot 的检查
- 退出注册表和命令行
- 输入 Windows 11 产品密钥，或选择版本
- 划分硬盘区域
- 按照指引完成 Windows 11 安装

# 完成安装

在基本设置完成和第一次启动后，关闭电源，将启动顺序调整为硬盘优先。