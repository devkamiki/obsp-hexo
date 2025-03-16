---
layout: post
cid: 22
title: Microsoft Account辅助邮箱验证bug:That doesn't match the alternate email associated with your account.
slug: microsoft-alternate-email-bug
date: 2025/02/25 21:56:00
updated: 2025/03/05 12:02:23
status: publish
author: 神木友希
categories: 
  - 日常
tags: 
  - microsoft
  - issues
desc: 
img: 
---


Bug描述: 格式为xx.abcd@example.org的邮箱是微软账号的辅助邮箱之一。试图验证辅助邮箱地址，输入地址后显示 "That doesn't match the alternate email associated with your account. The correct email starts with "xx".
Bug原因: 前端程序员太弱智。
解决方案: 把第三个字符`.`换成URL编码的`%2E`即可。

类似问题: 
>In my case, I only have two letters as an email name.
For example, if my alternative email is *** Email address is removed for privacy ***
Verify your email
We will send a verification code to ab*****@XYZ.org. To verify that this is your email address, enter it below.
When I key in *** Email address is removed for privacy ***, it always shows
That doesn't match the alternate email associated with your account. The correct email starts with "ab".

解决方案: 目前无解。

参考文献: [Alternate Email not recognized for Verification code](https://answers.microsoft.com/en-us/outlook_com/forum/all/alternate-email-not-recognized-for-verification/3cb094ef-d1da-4294-891f-0f4d8cd7b3a2)
