---
layout: post
title: "javascript的一些问题"
date: 2016-10-21
categories:
- javascript
- frontend
---

### 版本问题   
这里说的javascript版本问题不是指浏览器兼容性。其实对于开发来说，有了babel的话，根本不需要担心版本问题，我一样可以用最新的语法特性。   

但是问题是，每一次javascript的版本升级，不像是真正意义上的版本升级。诚然，每个版本都在解决一部分问题，但是各个升级后的版本看起来更像是一个新的方言。光导入就有AMD、commonjs和es6 import。平常用的时候可能感觉没啥问题，我能用es6的地方都是用import的，但是有些地方使用import是不能解决问题的，你需要使用require。比如在webpack使用react引入图片的问题，你只能使用require，而不能使用import！！！   

```jsx
 <Image src={require('../img/xxx.jpg')} />
```

当时各种场景都试过了，根本没有思考要使用require去做这件事，结果试了一天都没把一个破图片搞出来。。。最后换了require试了一下居然出来了。。。后来认为应该跟webpack的加载方式有关系，尽管babel是会把import翻译成require的，但是webpack是把图片hash过的，这里需要一层mapping吧。总之，看前端项目的时候，就是import中混合了零星的require。   

### javascript给开发人员带来了无限的想象   

此前，没有任何一款语言能像javascript一样，像瘟疫一样在世界各个角度，横向纵向的传播。从最早活跃在浏览器内部的js，到后台依托v8的nodejs，在到后来的桌面应用，甚至翻译成lua之后可以跑在嵌入式系统。一部分人极力推崇javascript是多么多么强大的语言，一部分人承认javascript是有问题的，并认为迭代开发升级是解决之道，另一部分人认为js。。。我就是最后一部分人，这语言坑太多了。真正能够掌握这门语言得记住多少pattern啊。   

github统计贡献代码量js常年排第一，为啥啊？还不是因为这语言问题太多。前端发展到现在还是围绕着html、css和js，到现在前端也没有一个说的出来的最佳实践。每一个新框架、库仅仅是尝试解决某一个问题，要想真正构建一个生产级别的环境，就是把各个解决单一任务的第三方依赖组织到一起。看起来这是一个一劳永逸的事情，但是这件事从来没有停止发展过。。。几乎每两个月就会有新的解决方案替代旧的解决方案。   

这不，最近facebook看不下去npm了，就自己搞了yarn。但是你确定yarn就足够好嘛。。。明天google会不会出个啥幺蛾子。。。不过版本管理这种东西让facebook这种公司维护，确实要比某某社区要好。   

最近在开发一个chrome extension，开发google的东西，还是用angular更合适。由于当时像用material ui，就用了angular material。这真是一个失败的决定。。。这个库跟react下的material-ui简直没法比。从文档组织到具体实现。当我开发一个功能时，需要光标自动聚焦到输入框中，结果内置的directive居然不支持。这个issue被好多人讨论。解决官方宣称现在要开发angular material 2，所以决定“close this issue”。怪angular 2咯？



