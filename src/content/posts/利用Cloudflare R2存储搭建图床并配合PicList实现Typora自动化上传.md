---
title: 利用Cloudflare R2存储搭建图床并配合PicList实现Typora自动化上传
published: 2025-03-10
description: ''
image: ''
tags: [开发,工具,配置]
category: '经验分享'
draft: false 
lang: ''
---
- 前置条件：1个正常的Cloudflare账号

## 1. 创建R2存储桶

首先进入你的[Cloudflare面板](https://dash.cloudflare.com)
在左侧菜单中选择R2对象存储，进入后点击创建

![](https://r2.c4lm0n.top/picture_bed/2025/03/e673371eb2a0de469e6db7f95b4b4747.webp)

输入你喜欢的名字命名你的存储桶，然后完成创建【**其他选项不更改**】

![](https://r2.c4lm0n.top/picture_bed/2025/03/8bffe9b83371557e880ff2830cfb5244.webp)
【**可选**】进入刚才完成创建的存储桶

![](https://r2.c4lm0n.top/picture_bed/2025/03/970a71c8e4fb3e2a43130906ab4e8fc0.webp)
加入你自己的域名从而避免公共域名访问的限速

![](https://r2.c4lm0n.top/picture_bed/2025/03/54a5d1b70f9aeeed5eebb9f4872cfa57.webp)

> ps:前提条件是你拥有自己的域名且该域名的DNS解析服务在Cloudflare下（可通过cf管理域名DNS）

## 2. 创建R2存储桶访问API

回到创建R2存储桶的界面，进入管理API

![](https://r2.c4lm0n.top/picture_bed/2025/03/c105c767fb18d26a01d7eea190fd3e46.webp)
点击创建API令牌

![](https://r2.c4lm0n.top/picture_bed/2025/03/7ac4aee02d152e2d015f7a2422780ded.webp)
API名称按个人喜好命名，然后按照下图所示设置

![](https://r2.c4lm0n.top/picture_bed/2025/03/5a82bef4d4b8abe9fdfa9bd328c5fdd8.webp)
完场创建后将会展示API令牌相关信息，记下备用
**切记！该页面仅显示1次，后续不可再次查看，请妥善包管相关信息**
**处于信息安全考虑，建议不要存储此页面信息，仅在本次配置时暂时保存以便使用**
**令牌应当遵行do one thing原则，有其他需要或者忘记令牌应当重新创建**

![](https://r2.c4lm0n.top/picture_bed/2025/03/2cf7bb9fa2d9faab34767a360adab8a3.webp)

## 3. 安装PicList

下载安装[PicList](https://github.com/Kuingsmile/PicList/releases/latest)【PicGo也可，但是需要自行安装S3插件，后续设置步骤相同】
如果打不开上面的超链接，可以使用这个[加速地址](https://release.piclist.cn/latest/PicList-Setup-2.9.8-x64.exe)【版本可能不是最新，不影响使用】
**安装过程简单就不贴图了，但是一定要记住自己的安装位置，后续会用到**
打开PicList后，进入如下位置

![](https://r2.c4lm0n.top/picture_bed/2025/03/2f42fabd6fc5c7357e8b0b6c4b249114.webp)
按照如图所示完成设置

![](https://r2.c4lm0n.top/picture_bed/2025/03/35001b83d4aa5bdd1cfc62ba36b1c275.webp)
点击确定保存，在返回的界面底部选择**设为默认**，至此PicList基本设置完毕

![](https://r2.c4lm0n.top/picture_bed/2025/03/c31fe3972e61e2d05d22fbb7a9752c5e.webp)

> 可以上传一张图片验证设置是否正确，不正确检查上诉步骤；更多高级设置自行按需探索

## 4. 设置Typora

打开Typora，依次点击文件-偏好设置，按下图进行设置

![](https://r2.c4lm0n.top/picture_bed/2025/03/2be09f85071149e445e36b5d9696e44b.webp)
完成后点击验证，看到成功反馈则所有设置正确，至此本篇教程结束（完结撒花*★,°*:.☆(￣▽￣)/$:*.°★* 。）
