---
layout: post
title: 在java应用中集成jython
date: 2017-01-01
categories:
- java
- jython
- jsr223
---

今天是新年第一天，但是上周欠的帐还没还完。   

上周在规则引擎的系统中加入了python部分的实现，即jython，碰到了好多坑。。。有种要死的感觉。。。其实系统之前已经集成了groovy，使用的实现是GroovyScriptEngine，就是把一段字符串形式的groovy脚本通过groovy classloader加载进来，相当于直接执行编译后的class文件。为了能平滑支持jython，实现改成jsr223的通用实现，即script engine，同时支持groovy和jython2.7（项目需要这个老版本。。。）。改完之后，jython各种不支持，然后就一点点填坑。最后就剩下最后一个问题了！本地可以正常运行，但是上到demo环境就不行了。。。然后会神奇的抛出一个空指针异常，说对应的engine没有找到。debug跟进去发现jython初始化engine的时候需要到一个叫Lib的地方import一些library，其中有一个“site”没有找到。然后就没法初始化这个jython engine。

我发现之所以，本地可以跑jython是因为跑spring boot的时候我是在intellij中用main函数的方式跑，这时候所引用的第三方jar是从.gradle/cache下读的。而通过gradle build，然后java -jar xxx方式启动的时候，jython会向这里找Lib：/Users/magiclane/git/dr/morpheus/api/build/libs/api-1.0.jar!/BOOT-INF/lib/jython-standalone-2.7.0.jar。 问题就出在路径上了。   

spring boot build称可执行的jar之后的结构是这样的：   
>example.jar
 |
 +-META-INF
 |  +-MANIFEST.MF
 +-org
 |  +-springframework
 |     +-boot
 |        +-loader
 |           +-<spring boot loader classes>
 +-BOOT-INF
    +-classes
    |  +-mycompany
    |     +-project
    |        +-YourClasses.class
    +-lib
       +-dependency1.jar
       +-dependency2.jar

而且java -jar xxx的时候，这些第三方jar也不会被打开。   

然而jython找Lib的默认位置是通过自己的找一个class的classpath定位的。   
```java
String urlString = URLDecoder.decode(rawUrl, "UTF-8");
urlString = urlString.replaceAll(escapedPlus, plus);
int jarSeparatorIndex = urlString.lastIndexOf(JAR_SEPARATOR);
if (urlString.startsWith(JAR_URL_PREFIX) && jarSeparatorIndex > 0) {
	// jar:file:/install_dir/jython.jar!/org/python/core/PySystemState.class
	jarFileName = urlString.substring(JAR_URL_PREFIX.length(), jarSeparatorIndex);
}
```

然而真实路径却是在那之前还有一段：**libs/api-1.0.jar!**
所以上述代码只截取了一段路径，而不是正确路径。我看网上有好几个帖子在讨论这件事，大致相同，如何在spring boot中正确引用jython-standalone.jar中的Lib。官方的文档也提到这一块，看完我就惊呆了，jython的文档居然说可以把jython-standalone.jar打开，再把我自己的代码放进去，然后再打包。并且说这是**the easiest way**。。。就这一个问题我就一直没找到好方法解决。然后就坐地铁回家了，地铁上又继续想这事，冥冥之中好像gradle可以做这件事，就重新看了spring boot文档中关于build的那块文档，发现requiresUnparck这个命令应该能有用。而且文档中说的jruby的问题和jython的问题很像：

> Most nested libraries in an executable jar do not need to be unpacked in order to run, however, certain libraries can have problems. For example, JRuby includes its own nested jar support which assumes that the jruby-complete.jar is always directly available as a file in its own right.
To deal with any problematic libraries, you can flag that specific nested jars should be automatically unpacked to the ‘temp folder’ when the executable jar first runs.

回家后跃跃欲试，我就把gradle改成了这样子：
```groovy
springBoot  {
    requiresUnpack = ['org.python:jython-standalone:2.7.0']
}
```

可是还是不行，依然没有找到正确的路径。
一直干到凌晨两点，脑子直接晕了。其实那个时候脑子已经跟屁股差不多了，根本没法思考。然后睡觉，8点多起床接着干。把版本号去掉，终于可以了。。。呃，关键时刻还是要信文档啊。

```groovy
springBoot  {
    requiresUnpack = ['org.python:jython-standalone']
}
```

这样就可以了。。。回头看了一眼文档，在api文档中，确实强调了：   
>requiresUnpack
A list of dependencies (in the form “groupId:artifactId” that must be unpacked from fat jars in order to run. Items are still packaged into the fat jar, but they will be automatically unpacked when it runs.)

现在的路径就正确了：
**/private/var/folders/py/fl2p5tnn79x4hm6p5djnh8_80000gn/T/api-1.0.jar-spring-boot-libs-688b9802-0e25-418c-ace5-169effa952c8/Lib**    

**/var/folders/py/fl2p5tnn79x4hm6p5djnh8_80000gn/T/api-1.0.jar-spring-boot-libs-688b9802-0e25-418c-ace5-169effa952c8/jython-standalone-2.7.0.jar/Lib**   

