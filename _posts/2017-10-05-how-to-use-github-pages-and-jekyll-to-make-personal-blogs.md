---
layout: post
title: "如何使用 github pages 和 jekyll 搭建个人博客"
date: "2017-10-05"
description: "如何使用 github pages 和 jekyll 搭建个人博客"
tag: [github pages, jekyll]
---

**前提假设**: 基本了解 git, github 是什么以及简单操作, 并有 github 账号。  

### github pages 简介以及建立页面
github pages 被设计为直接从 github 存储库托管个人、组织或项目页面。要了解更多不同类型的 github 页面站点, 是一个静态站点托管服务。  
在自己的 github 账号下建立对应的用户级别页面, [操作指引](https://pages.github.com/) 。

### jekyll 简介以及安装 (本文 jeykll 是在 windows 环境下安装)
jekyll 是一个简单的博客形态的静态站点生产服务。它有一个模版目录,其中包含原始文本格式的文档, 通过一个转换器(如   Markdown) 和 Liquid 渲染器转化成一个完整的可发布的静态网站, 可以发布在任何的服务器上。  
**安装 jekyll** ([操作指引](http://jekyll-windows.juthilo.com/))
- 安装 ruby 和 ruby devkit (后续使用 gem install 命令报错时, 可参考 [ridk install](https://github.com/oneclick/rubyinstaller2#using-the-installer-on-a-target-system) 相关)
- 安装 jekyll  
- 安装 rouge 或 pygments 用于高亮 (这一步可选)  
- 安装 wdm (在安装时遇到了 [error](https://github.com/oneclick/rubyinstaller/issues/276), 发现是因为 ruby 安装了 2.4 版, 而 ruby devkit 只支持到 2.3 版。 所以重装 ruby 和 ruby devkit)  
- 启动 jekyll 在本地查看效果 (如果在此步遇到了如下 [error](https://github.com/jekyll/jekyll/issues/5165), 采用高票答案尝试)
启动后在浏览器访问 **http://localhost:4000** 在本机查看效果 (应该看到 github pages 中创建的 index.html 页面)

### 使用 jekyll 主题模板
- 使用 github pages 提供的默认模板, [操作指引](https://help.github.com/articles/creating-a-github-pages-site-with-the-jekyll-theme-chooser/) 。  
- 使用 github 上其他人的模板, [模板列表](http://jekyllthemes.org/) 。
- 自己造轮子写主题  
因为才接触还不知道怎么写模板, 最好的办法就是 fork 其他大神的漂亮模板啦 (逃...  

### TODO
- 如何将 github pages 绑定到自定义域名
- 如何造轮子写主题
- 添加评论,打赏等功能
- 图床, 图片压缩 (目前博客量很小,图片可存在项目中引用相对路径)
- ...


>**参考:**  
[github pages](https://pages.github.com/)  
[jekyll](https://jekyllrb.com/)  
[windows 运行 jekyll](http://jekyllcn.com/docs/windows/#installation)   
[run jekyll on windows](http://jekyll-windows.juthilo.com/)  
[jekyll thumes](http://jekyllthemes.org/)  
[有哪些简洁明快的 jekyll 模板](https://www.zhihu.com/question/20223939)  
[using a custom domain with github pages](https://help.github.com/articles/using-a-custom-domain-with-github-pages/)  
**TL;DR:**  
[如何搭建一个独立博客——简明 github pages与 jekyll 教程](http://www.cnfeat.com/blog/2014/05/10/how-to-build-a-blog/)  
[github pages + jekyll 创建个人博客](http://www.jianshu.com/p/9535334ffd54)  
[利用 github pages 快速搭建个人博客](http://www.jianshu.com/p/e68fba58f75c)  
[github pages + jekyll 独立博客一小时快速搭建&上线指南](http://playingfingers.com/2016/03/26/build-a-blog/)  
**感谢:**  
[hywelzhang](https://hywelzhang.github.io), [leopardpan](https://leopardpan.github.io/)
