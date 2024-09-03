---
title: Hey Vim! I'm back to you, again!
layout: post
date: 2022-07-12
tag:
- vim
---

过去一段时间主力开发语言是rust和c，当然可能会写一些python/js。日常工作使用vs code作为开发工具，还是比较省心的。
但是最近发现vs code越来越卡了。由于工作需要，经常需要开4、5个项目，来回切着看代码、调试。CPU直接起飞了！
这不得不让我重新想想，vs code是不是真的适合我了。vs code最大的优点是省心，不需要折腾。我也确实很久很久没折腾工具了0.0

所以上周折腾了一下，还是很开心的！

我开始重新使用vim，其实不是vim，是neovim。一个提供异步支持的neovim，倒逼vim进步的项目。当然vim 8也支持异步了，所以我理解这两者差距不大。
值得一提的是，nvim支持使用lua写插件，好多lua写的插件都跑的好快。

最令我开心的是一整套的工具连，用起来十分得心应手，在teminal里完成所有的编辑工作，根本没有碰鼠标的必要。

- alacritty - Alacritty - A fast, cross-platform, OpenGL terminal emulator
- zsh
- tmux
- nvim

我把相关配置放到里[github](https://github.com/magicalne/config).

alacritty只做一件事——terminal。没有tab、window等等其他概念。所有配置的修改都是通过yml文件。一个很大的亮点是支持vi-mode。
所以用户需要使用类似tmux的软件来管理window。Oh tmux. So great!

zsh的插件我没有用很多，主要使用了zsh-autosuggestion。这个插件师从fish，我用了好久的。

没想到啊，隔了好几年，还是回到了原点...
