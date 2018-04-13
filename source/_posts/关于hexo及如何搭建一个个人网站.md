---
title: 关于hexo
date: 2018-04-13 23:27:32
tags: hexo blog
---
![hexo](https://hexo.io/logo.svg)

hexo是一个致力于搭建blog的框架。

```
A fast, simple & powerful blog framework
```
本文不涉及如何使用hexo搭建一个blog，毕竟网上的教程非常多。
大多教程都是使用hexo与github pages联合使用。
在我搭建过程有遇到两个地方比较难解



>1. hexo怎么和github pages结合
>2. github pages 怎么才能创建出来（使用 xxx.github.io 进行访问）




第2个问题比较好解，先说第二个。
github pages创建有一定规则。网上的教程会告诉你需要先搭建个仓库（New repository）,但创建完后，访问xxx.github.io总是提示找不到😂😂😂。原因是在创建repository时，创建的repository名称必须和你的github名称一致。![repo](/images/repo.jpg)

然后进入到该库中，去settings中，往下拉就可以看到你已经成功创建pages。
github的page可以自定义的👍，当然你也可以购买个域名指向你的github pages。
![settings](/images/settings.jpg)

第1个问题在理解了github pages时，才得到了答案。
github pages会load你在这个库提交的html。
![html](/images/html.jpg)
hexo的功能可以把*.md文件转成html及相关css等。通过一些插件直接提交到github里。
至此，你使用markdown这样工具编译的md文件，可以使用hexo转换html，即可实现一个自己的blog。