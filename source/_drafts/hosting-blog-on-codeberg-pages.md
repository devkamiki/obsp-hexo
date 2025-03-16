---
title: 在Codeberg Pages上使用Hexo搭建博客
tags:
  - codeberg
  - pages
  - staticsite
  - blog
---
Numerus Fixus 项目的考试终于结束了，发挥得比较一般，没什么把握。放榜之后再整理一下自己的笔记和经验分享给大家吧（虽然感觉几乎没有人用得到，不过万一呢）。
放榜之前都是空闲时间，顺手把旧文章通过[Tp2MD 插件](https://github.com/AlanDecode/Typecho-Plugin-Tp2MD)导出了，写一写obsp 的Hexo 镜像搭建过程吧。
Typecho 的部署相对来说简单一些，可以直接docker compose，数据库也每天都有备份，但是用久了感觉还是有点不顺手，也许直接从Markdown生成HTML的静态站点生成器更适合我一点。静态站点的内容可以托管在GitHub Pages、Gitlab Pages、Bitbucket Cloud和Codeberg Pages等，省去了挑选和维护服务器的烦恼，从写作文章到发布和查看效果的过程也更快速便捷。更重要的是，我发现Typecho 站点被搜索引擎抓取的时候似乎很难抓到文章标题，常常将正文的内容作为标题收录，这就导致了通过搜索关键词发现obsp 的读者往往会因为搜索引擎抓取的标题太奇怪而不愿意点进来，而Hexo 站点就没有这种缺点。Hexo 的缺点大概是每次重装系统/换新设备，都要重新配环境。其他比较出名的静态站点生成器例如Hugo 和Jekyll 我还完全没有尝试过。
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
Hexo 官方文档没有详细介绍站点目录中各个文件/文件夹的作用，新手可能会一头雾水：我的文章应该放在哪里？站点的设置文件在哪里？这个文件夹就是我的站点源码吗？