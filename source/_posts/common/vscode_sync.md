---
title: vscode同步配置
date: 2017-10-30 15:19:32
categories:
reward: false
tags:
     - blog
     - vscode
---

## Syncing
可以使用Syncing这个vscode的插件实现，该插件是通过github gist来实现的。插件可在vscode应用市场内找到。

[Syncing安装](https://marketplace.visualstudio.com/items?itemName=nonoroazoro.syncing)

## gist
是github的一个分享代码片段的功能，可以运用gist作很多有趣的事情。

[gist申请](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)

注意：
+ 为了安全考虑，gist只有在第一次生成的时候才会显示token值，所以请牢记申请的gist token。
+ gist申请方式也可参考Syncing的插件说明。

<!--more-->

## 安装
+ 申请github gist，并记录tokens
+ 安装vscode Syncing插件
+ F1打开vscode cmd，查找upload setting，输入gist token
+ 之后就可以直接使用update和download同步功能