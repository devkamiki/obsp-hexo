---
title: 在Codeberg Pages上使用Hexo搭建博客
tags:
  - codeberg
  - pages
  - staticsite
  - blog
date: 2025-03-18 22:16:28
---

## 前言

Numerus Fixus 项目的考试终于结束了，发挥得比较一般，没什么把握。放榜之后再整理一下自己的笔记和经验分享给大家吧（虽然感觉几乎没有人用得到，不过万一呢）。
放榜之前都是空闲时间，顺手把旧文章通过[Tp2MD ](https://github.com/AlanDecode/Typecho-Plugin-Tp2MD)插件导出了，写一写obsp 的Hexo 镜像搭建过程吧。
Typecho 的部署相对来说简单一些，可以直接docker compose，数据库每天备份，但是用久了感觉还是有点不顺手，尤其是发布和修改文章非常麻烦，每次发现错别字都要手动登录后台捉虫。也许直接从Markdown生成HTML的静态站点生成器更适合我一点。静态站点的内容可以托管在GitHub Pages、Gitlab Pages、Bitbucket Cloud和Codeberg Pages等，省去了挑选和维护服务器的烦恼，从写作文章到发布和查看效果的过程也更快速便捷。更重要的是，我发现Typecho 站点被搜索引擎抓取的时候似乎很难抓到文章标题，常常将正文的内容作为标题收录，这就导致了通过搜索关键词发现obsp 的读者往往会因为搜索引擎抓取的标题太奇怪而不愿意点进来，而Hexo 站点就没有这种缺点。Hexo 的缺点大概是每次重装系统/换新设备，都要重新配环境。其他比较出名的静态站点生成器例如Hugo 和Jekyll 我还完全没有尝试过。
这篇文章主要聚焦使用Codeberg Pages的静态站点托管服务来托管Hexo 生成的页面打造博客的过程。Hexo 的官方文档对于Git部署说得也很详细，不过我会加入一些自定义和个性化的过程，例如使用主题、利用Giscus 连接GitHub repo discussions 作为评论区等。

**以下操作在基于Arch Linux的发行版Manjaro Linux（内核：Linux 6.12.17-1-MANJARO）上完成，操作系统为64位**

## 安装必要依赖

Git 和 Node.js，由于自带，此处不再赘述，其它操作系统/发行版可以参考[Hexo官方文档](https://hexo.io/docs/#Install-Git)。

## 安装Hexo

Hexo 的安装非常简单：`npm install -g hexo-cli`
安装完成后，我们可以新建一个站点，每一个站点的设置和数据会以文件夹的形式储存，所以我们在你期望的根目录处进行init：
```
$ hexo init <folder>  
$ cd <folder>  
$ npm install
```
这样，一个基本的站点目录结构就完成了。
新手可能会一头雾水：我的文章应该放在哪里？站点的设置文件在哪里？这个文件夹就是我的站点源码吗？
其实，Hexo生成的静态站点现在还没有出现，我们的站点文件夹中存放的是用来生成HTML文件的资源。我们可以通过使用Hexo的各种命令或者直接读写站点文件夹中的文件来修改资源，从而修改最终生成的页面。
文件夹应该是这个结构：
```
.  
├── _config.yml  
├── package.json  
├── scaffolds  
├── source  
|   ├── _drafts  
|   └── _posts  
└── themes
```
很好理解，`_config.yml`是配置文件，`package.json`是应用程序的信息，`scaffolds`文件夹中存放模板，`source`存放的则是决定最终生成页面的文章内容的资源，包括尚未完成的草稿`_drafts`和已经准备好渲染和发布的`_posts`。
以Observer's Space Hexo站为例，我用Obsidian打开`~/obsp/source`文件夹作为图谱，在图谱下面，可以看到我正在写的这篇《在Codeberg Pages上使用Hexo搭建博客》归属在`_drafts`文件夹里，而大家在Observer's Space Hexo站上能看到的历史文章都归属在`_posts`文件夹里。当我们执行生成命令的时候，`_drafts`文件夹里的文章不会生成页面，当你准备好将它们发布的时候才会被移动到`_posts`文件夹，参与网页的生成。生成的HTML文件会被放在`public`文件夹里，当我们使用GitHub Pages/Codeberg Pages/Bitbucket Cloud的静态站点托管服务时，我们做的主要是通过Hexo命令使用Git来将本地的`public`文件夹与远程仓库进行同步，Pages服务通过读取仓库中的内容展示网站。（GitLab Pages的部署略有不同，GitLab用户可以参考Hexo和GitLab Pages的官方文档，此处不再赘述。）
Themes存放的是主题文件的配置，我们现在先不管它。

## 修改配置文件

现在我们需要做的事是修改配置文件。仍然以Observer's Space为例，由于`_config.yml`太长了，我只截取重要的部分进行说明：
```
title: Observer's Space Mirror #站点的标题
favicon: https://s21.ax1x.com/2025/02/11/pEn6KsJ.png #访问网站时显示的头图，我选用了可以免费使用的Google Noto Emoji中ringed planet的Android 10版本，新版设计可以参考https://emojipedia.org/ringed-planet#designs
avatar: https://s21.ax1x.com/2025/02/11/pEn6KsJ.png #网站的头像，细心的朋友可能发现我偷懒了，和头图用的是同一张嘿嘿
subtitle: '宇宙中的观星者' #副标题
description: 'This is a sister website of obsp.de. (Still in beta)' #描述
keywords: #关键词，懒得想
author: Yuki Kamiki #神木友希的大名
language: en
timezone: 'UTC'

## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
url: https://hexo.obsp.de #这个可以设置成你等下要绑定的自定义域名，就像obsp一样，如果你想用Codeberg给的域名，就请设置成https://USERNAME.codeberg.page。错误的域名设置将会导致CSS和JavaScript资源加载失败，只能访问到文字内容。

#。。。有一群乌鸦飞过
#。。。有一群乌鸦飞过

deploy: #涉及到使用Hexo自动部署命令的配置
  type: 'git' #我们使用git的方式进行自动部署
  repo: git@codeberg.org:kamiki/obsp.git #Repo的地址，https或者ssh都可以。我的 pages 仓库拿去做别的内容了（欢迎大家访问~），所以给 obsp 创建了一个新仓库。如果你要用Codeberg Pages 并且在 Codeberg 提供的域名根目录 https://USERNAME.codeberg.page 访问的话就应该是git@codeberg.org:USERNAME/pages.git
  branch: main #跟踪远程仓库的分支
  message: updates #commit时的提交信息
```
啊，配置文件搞定了！不要忘记保存退出哦！

## 生成静态站点

你可以使用`hexo new [layout] xxx`来新建一篇文章。
当你指明 layout 是draft/page/post 的时候，hexo 会将生成的 md 文件存放在`source`目录下对应类型的文件夹中，如果省略 layout 参数，那么默认生成的是 post 。
如果还没想好写什么，没关系，打开 `source/_posts`，会发现那里面并不是空白的，我们在执行`hexo init`的时候自动生成了一个 md 文件，它可以作为博客第一篇文章的资源。
执行`hexo g`，会发现生成了一个`public`文件夹。执行`hexo deploy`进行部署，会将这个文件夹中的内容发布到刚才在`_config.yml`中填写的仓库地址。
> 等一下……！好像不太对……
## 发布到Codeberg Pages

第一次用git的小伙伴可能不知道怎么连接远程仓库，~~请善用搜索引擎~~ 和平时连小鸡的方式差不多，用公私钥或者用户名密码都可以。什么你说你没有连过小鸡？~~请善用搜索引擎~~ 执行`ssh-keygen -t ed25519 -C "your_email@example.com"`，然后你就会在`~/.ssh`找到公私钥，公钥是那个文件名带 pub 的，私钥是不带的，把公钥内容上传到[ Codeberg 设置中 ](https://codeberg.org/user/settings/keys)，然后再执行`hexo deploy`即可。
## Enjoy！

稍等片刻（Codeberg 正在码头给你整 SSL 证书），即可在`https://USERNAME.codeberg.page/`查看你的 landscape 主题 Hexo 博客。

## 绑定自定义域名

把你的域名 CNAME 解析到`username.codeberg.page`，然后在仓库里新建一个`.domains`文件，输入你想要的域名`example.org`，再等一会（Codeberg 正在码头给你整 SSL 证书）即可通过自定义域名访问。
什么你说 root 不能有 CNAME 记录？~~你怎么不托管到能做CNAME拉平的NS提供商~~ 也没问题，A记录解析到`217.197.91.145`，AAAA记录解析到`2001:67c:1401:20f0::1`，然后TXT记录`username.codeberg.org`即可。

## 主题配置（以本站为例）

本站采用的是 [Stellar](https://xaoxuu.com/wiki/stellar/) 主题，感谢 xaoxuu 大佬开发和开源！
因为我比较懒，所以采用了稳定版安装：`npm i hexo-theme-stellar`
找到`_config.yml`文件中的`theme: landscape`改成`theme: stellar`然后重新运行`hexo g`和`hexo deploy`即可。

## 自定义字体（以本站为例）

本站用的是霞鹜文楷（无衬线体、衬线体）和 PT Mono（等宽）。
新建一个专门给 Stellar 主题用的配置文件`_config.stellar.yml`，写入以下内容：
```
inject:
  head:
    - <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/lxgw-wenkai-screen-web/style.css" />
    - <link rel="preconnect" href="https://fonts.bunny.net">
    - <link href="https://fonts.bunny.net/css?family=pt-mono:400" rel="stylesheet" />
style:
  font-family:
    logo: 'LXGW WenKai Screen'
    body: 'LXGW WenKai Screen'
    code: 'PT Mono'
    codeblock: 'PT Mono'
```
注意 Stellar 仓库里的`_config.stellar.yml`示例使用了单引号内套双引号`'"LXGW WenKai Screen",...'`，实测这样子无法应用字体，去掉双引号即可。

## Giscus 评论区

和 Typecho 不同，Hexo 默认是没有评论区的，而大部分博主还是希望能够听到读者的反馈，那么就有了各种各样的评论插件。Giscus 使用 GitHub 仓库的 Discussions 功能，按照 https://giscus.app/zh-CN 上的指导创建好公开仓库、安装 giscus app、启用 Discussions功能后，把你的用户名/仓库名复制粘贴进去（很奇怪，手动输入会提示无法连接，复制粘贴就可以，这个 bug 可以复现，[有人遇到了同样的问题](https://jachinzhang1.github.io/2025/02/04/hexo%E5%8D%9A%E5%AE%A2%E6%B7%BB%E5%8A%A0%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F/)）。
然后在 Stellar 配置文件里写入：
```
comments:  
  giscus:  
    enable: true  
    repo: <YOUR_GITHUB_USERNAME/YOUR_REPO_NAME>  
    repo_id: <YOUR_REPO_ID>  
    category: <YOUR_CATEGORY_NAME>  
    category_id: <YOUR_CATEGORY_ID>  
    mapping: pathname  
    reactions_enabled: 1  
    emit_metadata: 0  
    input_position: bottom  
    lang: en  
    loading: lazy  
    crossorigin: anonymous
```
重新运行一下`hexo g && hexo deploy`即可。