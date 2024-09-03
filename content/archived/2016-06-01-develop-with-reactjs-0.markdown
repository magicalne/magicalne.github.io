---
layout: post
title:  "Develop with reactjs -- part 0"
date: 2016-06-01
categories:
- frontend
- react
- redux
- ace
- material-ui
---
### 入坑
组里躺着一个裸体的规则引擎，已经一年多了，木有UI，程序员用起来都费劲的那种...年前用nodejs写了一个CMD工具去CRUD规则，然并卵。一个多月前感觉已经有必要开发一套UI了，否则项目无法推动，新增修改规则全靠我人力搬运，实在手累，然后就有了这篇文章...

### 技术选型
这个前端项目很明显，就是那种“一个人的项目”。一开始就想好了，既然一个人的项目，那一定要写的爽！技术必须用最新的。然而毕竟是前端项目，虽然对外号称本人是准全栈工程师，然而前端这种博大精深的东西怎么是我这中渣渣掌握的了得...特别是我这种懒人，不想写HTML和CSS:(

选型的时候，前端框架基本没跑——***reactjs***无疑，好奇好久了，终于可以一探究竟了！可是webpack好难学，不会。不会webpack，就不会自己搭项目框架，不会搭框架，就没法写代码...不过好在github上已经有很多boilerplate了。这里我使用了[mxstbr/react-boilerplate](https://github.com/mxstbr/react-boilerplate/tree/v3.0.0) 这里不得不说一下，作者是来自于“offline-first foundation”，听名字也懂得~但是由于我的项目不需要offline，所以这个东西对于我来说是累赘，而且还引起了一个bug，这个bug是在我用react-router，实现不跳转刷新页面时出现的，除非使用强制刷新，否则子页面是无法刷新出来的，希望是我使用不当吧，反正我是把offline删掉了。

框架中使用了redux来替代flux，所以***redux***也没跑了。虽然当时还不懂啥是redux...

使用***webpack***作为打包工具。

不喜欢写前端的最大原因就是不想学习CSS的实现机制...也不喜欢写CSS。但是对[material ui](https://www.google.com/design/spec/material-design/introduction.html)很是喜欢...自己肯定没能力写出那么漂亮的组件。So，随便在网上搜了搜，我的天，material ui实现的现成框架不要太多。想想我也就是半年没写前端，竟然出了那么多东西...于是我就挑选了一套实现material ui的react component——[material-ui](http://www.material-ui.com/#/)。可以先说结论，值得使用，99%的体验都很棒，但是框架自己也有很多bug，看你怎么取舍了，大都不那么致命，也都有hack的办法。比较恶心的是，文档过于先进，代码没实现的功能在文档中是实现了的...严重怀疑写文档的和写代码是不是一批人...当时我用的时候还是v0.14.X，没俩月就出v0.15了。更新还是很快的。

由于项目中需要写代码，所以需要一个编辑器，那就必须要使用大名鼎鼎的[Ace](https://ace.c9.io/)！真的很强大！本身由于Ace不是react component，如果直接使用还要自己写wrapper，像我这么懒的人怎么可能会写wrapper呢...而且我也不会写啊...于是，我又在github上找到了[react-ace](https://github.com/securingsincity/react-ace)，算是ace的功能子集吧，但是已经够我的项目用了。使用过程中由于我没有仔细阅读文档，进而写出了一个bug，以为是react-ace的bug，然后去社区提问，结果没多久就有热心人指出了我的使用问题。社区还是很活跃的，值得信赖。

So，大体上就是这样的，技术栈：reactjs，redux，webpack，material-ui，ace。其中还使用了es6,所以还涉及babel。后面再说～
