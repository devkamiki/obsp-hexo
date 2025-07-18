---
title: 电子邮件的“自动发现”
draft: "true"
date: 2025-07-18 20:26:39
tags:
  - email
  - autodiscover
  - dns
categories:
  - 日常
---
传统电子邮件客户端采用的自动发现通常通过查询域名 DNS 中特定的 SRV 记录来完成，这些 SRV 记录的标准格式如下：

```text
_submission._tcp     SRV 0 1 587 mail.example.org.
_submissions._tcp     SRV 0 1 465 mail.example.org.
_imap._tcp     SRV 0 1 143 imap.example.org.
_imaps._tcp    SRV 0 1 993 imap.example.org.
_pop3._tcp     SRV 0 1 110 pop3.example.org.
_pop3s._tcp    SRV 0 1 995 pop3.example.org.
```

其中 submission 和 submissions 对应的是 SMTP 服务器的 STARTTLS 和 SSL/TLS 链接，imap 和 imaps 对应 IMAP 服务器的非 TLS 连接和 SSL/TLS 连接，pop3 和 pop3s 对应 POP3 服务器的非 TLS 连接和 SSL/TLS 连接。

可以通过设置 SRV 记录的域名为 `.` 来明确表示该域名不支持某一特定协议的连接，并且可以通过设置 SRV 记录的优先级来向客户端传递自动配置时优先使用的协议。

Thunderbird 采用的 autoconfig 则是通过查询 `https://autoconfig.example.org/mail/config-v1.1.xml?emailaddress=xxx@example.org` ，返回配置的 XML 文件实现的。例如，在浏览器中访问 https://autoconfig.disroot.org/mail/config-v1.1.xml?emailaddress=example@disroot.org, 会返回如下记录：

```xml
<!--
 autoconfig for Thunderbird as explained here:
     https://developer.mozilla.org/en/Thunderbird/Autoconfiguration#Configuration_server_at_ISP
     looking in path: http://autoconfig.disroot.org/mail/config-v1.1.xml 
-->
<clientConfig version="1.1">
<emailProvider id="disroot.org">
<domain>disroot.org</domain>
<displayName>disroot.org</displayName>
<displayShortName>disroot.org</displayShortName>
<incomingServer type="imap">
<hostname>disroot.org</hostname>
<port>993</port>
<socketType>SSL</socketType>
<authentication>password-cleartext</authentication>
<username>%EMAILLOCALPART%</username>
</incomingServer>
<outgoingServer type="smtp">
<hostname>disroot.org</hostname>
<port>465</port>
<socketType>TLS</socketType>
<authentication>password-cleartext</authentication>
<username>%EMAILLOCALPART%</username>
</outgoingServer>
</emailProvider>
</clientConfig>
```

如果你曾经使用 [Migadu](https://migadu.com) 提供的邮件托管服务，那么你可能会遵照 Migadu 的指引为你的域名添加如下一条 DNS 记录：

| Type  | Host(name)     |              | Value (Destination)        |
| ----- | -------------- | ------------ | -------------------------- |
| CNAME | **autoconfig** | .example.org | **autoconfig.migadu.com.** |

这条 DNS 记录是为了让你在使用自定义域名的时候让 Thunderbird 能够正确请求到位于 Migadu 网站上的 XML 记录。

Thunderbird 并不仅仅依靠 autoconfig 确定邮件服务器采用的配置。如果 autoconfig 发现失败，Thunderbird 会尝试 `https://www.example.org/mozilla.xml` 以及传统的 SRV 记录。Mozilla 的数据库中包含了大部分常用邮件域名的服务器配置，如果上述方法均失败，Thunderbird 会尝试从 Mozilla 服务器寻找配置。如果依然失败，Thunderbird 会开始启发式推测配置，依次尝试 imap.域名、pop.域名、pop3.域名、smtp.域名和 mail.域名，并为每个地址测试常见的 2-3 个端口。同时检查 SSL 是否可用，以及服务器在 CAPABILITIES 中声明的认证算法等信息。

新版的 Microsoft Outlook 能够使用特殊的 SRV 记录发现配置，格式为：

```text
_autodiscover._tcp   SRV 0 1 443 autodiscover.example.org
```

虽然采用了 SRV 记录的格式，但 autodiscover 本质上是通过 http 请求完成的，因此大部分服务商会设置该记录的端口号为 443，这样 Outlook 就会直接请求 `https://autodiscover.example.org/autodiscover/autodiscover.xml`。

你也可以通过 CNAME 记录来实现这一点，例如 [NameCrane](https://namecrane.com) 会建议用户添加这样一条 DNS 记录：

| Type  | Name/Host    | Value              |
| ----- | ------------ | ------------------ |
| CNAME | autodiscover | eu1.workspace.org. |

这样一来，Outlook 在请求 `https://autodiscover.example.org/autodiscover/autodiscover.xml` 时就会被重定向到 `https://eu1.workspace.org/autodiscover/autodiscover.xml`，不过需要注意的是，你会发现，在 curl 这一链接（需要使用用户名和密码登录）后并不会出现 XML 文本，而是出现这样的页面：

```html
<html>
        <header>
        <title>Z-Push ActiveSync</title>
        </header>
        <body>
        <font face="verdana">
        <h2>Z-Push - Open Source ActiveSync</h2>
        <h3>GET not supported</h3> 
        <br><br>
        More information about Z-Push can be found at:<br>
        <a href="http://z-push.org/">Z-Push homepage</a><br>
        <a href="https://z-push.org/download/">Z-Push download page</a><br>
        <a href="https://github.com/Z-Hub/Z-Push/issues">Z-Push Bugtracker</a><br>
        <a href="https://github.com/Z-Hub/Z-Push/wiki">Z-Push Wiki</a> and <a href="https://github.com/Z-Hub/Z-Push/wiki/Roadmap">Roadmap</a><br>
        <br>
        All modifications to this sourcecode must be published and returned to the community.<br>
        Please see <a href="http://www.gnu.org/licenses/agpl-3.0.html">AGPLv3 License</a> for details.<br>
        </font face="verdana">
        </body>
```

这是因为 NameCrane 的服务器采用了 ActiveSync 协议的开源实现 Z-Push。ActiveSync 不支持采用 HTTP GET 方式直接获取记录，而是需要通过兼容的客户端获取。

这意味着 NameCrane 提供的 autodiscover 只有支持 ActiveSync 的客户端（例如 Outlook 和 emClient）才能用，而不包括普通的 IMAP/SMTP 等协议的自动发现。我在 Claws Mail 无法发现域名邮箱的配置，在 Thunderbird 上的发现我认为是通过启发式推测实现的，因为 Thunderbird 发现的邮件服务器配置是 `mail.example.org` 而不是 `eu1.workspace.org`. 为了保证能在 Thunderbird 上自动发现，NameCrane 会指导用户将 mail.example.org CNAME 指向 `eu1.workspace.org` 或 `us1.workspace.org`。