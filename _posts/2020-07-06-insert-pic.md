---
layout: post
title:  "Jekyll博客中如何插入图片？"
date:   2020-07-06 14:29:24
categories: 博客管理
tags: 博客 图片 
---

* content
{:toc}
关于如何便捷地管理博客的图片





### Jekyll博客中如何插入图片？

该博客原模板中在about模块一直显示一张无法加载的图片，以为是部署在github服务器上网络较差，实在是碍眼，决定用github图片仓库中的图片链接代替它，但是仅替换原来链接的话仍然无法加载，遂尝试了以下几种方式：

1. 因为about模块的构建是基于md文档的，所以尝试用markdown语法插入图片链接，此时用的是url链接，结果依旧加载失败
2. 跟刘老师提了一嘴，她跟我说右键通过新标签页打开图片，这个链接在原url链接的基础上新增了raw=true参数，然后用html的image标签插入该链接，最后加载成功
3. 第二天反复打开该页面的时候又加载失败，在网上搜寻了一些资料，最终方案为：在博客的根目录下"G:\myblog\XiaoY-JOJO.github.io"，新建一个images文件夹，专门用来存储博客中的图片，然后用markdown语法插入相对路径"/images/blog.jpg"，最终成功加载。

![](/images/insert.png)

{% include comments.html %}