---
layout: post
title: "在 github 上如何选择一个开源许可证"
date: "2017-10-04"
description: "在 github 上的开源代码如何选择一个开源许可证"
tag: github
---

### 简单宽松的
MIT许可证是一个宽松的、简明扼要的许可证,只要用户在副本中提供了版权声明和许可声明,就可以拿你的代码做任何事情,你也无需承担责任。  
使用该许可证的项目: jQuery, .NET Core, Rails
### 关心专利
Apache 2.0许可证是类似于MIT许可证的一个宽松的许可证,但它同时还包含一个贡献者向用户提供专利授权相关的条款。  
使用该许可证的项目: Android, Apache, Swift
### 关心共享改进
GNU GPLv3许可证是一种公共版权的许可证,要求任何贡献代码或衍生工作的人以相同的条件来提供源代码,并且也提供了贡献者向用户提供专利授权相关的条款。
使用该许可证的项目：Bash, GIMP, Privacy Badger
### 项目不是代码
开源软件许可证可用于非软件作品,通常也是最好的选择。尤其是当涉及到的作品可以作为源被编辑和翻译时,例如: 开源硬件设计,选择一个开源许可证。
**数据，媒体，等**
CC0-1.0, CC-BY-4.0, CC-BY-SA-4.0 开源许可证可用于数据集到录像等非软件载体。注意 CC-BY-4.0 和 CC-BY-SA-4.0 不能够用于软件
**文档**
任何开源软件许可证活着对于媒体的开源许可证都可以应用于软件文档, 如果对软件和软件文档使用了不同的许可证, 可能要注意文档中的源码示例是使用软件对应的开源许可证。
**字体**
SIL Open Font License 1.1 可以使字体开源, 但是允许字体在其他作品中免费使用。
**混合项目**
如果项目包含了软件和其他载体, 可以使用多种许可证, 只要你能够明确项目的哪一部分使用哪一许可证。
### 更多选择
github 给予了更多开源许可证的选择, 并且详细注明了权限, 条件, 闲置等。详见 [Licenses](https://choosealicense.com/licenses/)
### 无许可证
github 推荐项目使用一个开源许可证。详见 [No License](https://choosealicense.com/no-license/)

### 快速选择
github 上新建仓库时可选择开源许可证模板, 已有仓库可在根目录下新建LICENSE文件也可选择模板(.gitignore 也可相同操作)
乌克兰程序员 [Paul Bagwell](http://paulmillr.com/), 画了一张分析图说明应该怎么选择。
![Image](/images/posts/2017-10-04-how-to-choose-an-open-source-license-on-github/1.png)

>**参考:**  
[wtfpl](http://www.wtfpl.net/txt/copying/)  
[choose-an-open-source-license](https://choosealicense.com/)  
[simple-description-of-popular-software-licenses](http://paulmillr.com/posts/simple-description-of-popular-software-licenses/)    
[如何选择开源许可证](http://www.ruanyifeng.com/blog/2011/05/how_to_choose_free_software_licenses.html)  
[如何为你的代码选择一个开源协议](http://www.cnblogs.com/Wayou/p/how_to_choose_a_license.html)  
[开源许可证都有什么区别,一般开源项目用什么许可证](https://www.zhihu.com/question/28292322)  
