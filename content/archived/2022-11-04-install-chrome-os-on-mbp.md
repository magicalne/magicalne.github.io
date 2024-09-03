---
title: ChromeOS Flex, 让我的macbook pro 2013重获新生
layout: post
date: 2022-11-04
tag:
- chromeOS
---

我人生第一台MBP在2015年买的——macbook pro 2013...

这电脑很多年都没怎么用了。特别是这几年开始远程办公，主要都是用家里的Desktop。半个月前公司组织outing，我还是把它带上了。很尴尬，期间出了bug，而且还需要我来修。。。
用这个2013款的mac实在太憋屈了，build c/rust太难为她了。本来是想着挂在咸鱼回回血，等着入国产arm/riscv笔记本。没想到，生活处处有惊喜！听了我司前CTO的podcast: [ #95 - 用 Chromebook 做开发是什么样的体验？ ](https://teahour.fm/95)，我的mbp 2013又活了。

本来他们是讨论的chromebook，以及瘦客户端。chromebook就是一个输入输出的设备，计算都在远端完成。可以是ssh连家里的server，也可以是连aws开的机器。开发环境需要能快速构建。所以chromebook就是一个很良好的选择。
当天就就下了个chromeOS flex镜像，直接boot！体验就是，安装很快，UI体验也很好，没有原生app，只有浏览器。还是有点点卡的，不过问题不大。

好巧不巧，当天晚上又失眠了，躺到凌晨4点怎么都睡不着，索性起来了。通宵身体还是不舒服的，也不想坐在电脑前。就开始把玩chromeOS，躺在沙发上，ssh连上desktop，开始干活! 
那天晚上也就写了个测试，但是整体上把平常需要用到的地方都用到了，也算把这套流程打通了。

有两个地方体验很差：
- chromeOS无法安装nerd font，导致ssh看到的字符不是等宽的，所以光标移动会有一个很大的误差，很影响开发体验。
- 不能从host直接复制到ssh server的剪切板。这是一个老问题了，不好解决，也影响不大。

nerd font有两个解决办法
- 可以在本地hack，chromeOS的terminal也是html写的，可以直接改css。这个是最快的，也实验过。
- 再就是整理一个精简的vimrc，去掉所有fancy的功能，当ssh上去使用精简vimrc，平时使用完全版。

