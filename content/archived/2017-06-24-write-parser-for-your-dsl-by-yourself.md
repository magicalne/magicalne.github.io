title: 如何为自己的DSL写parser
layout: post
date: 2017-06-18
tag:
- java
- DSL
- parser
- parboiled

---

### 什么是DSL？

DSL——Domain specific language，领域语言。

DSL在规则引擎中有广泛的应用，但其实不单单是规则引擎，很多地方都有DSL的身影。比如最简单的JSON，SQL，properties配置文件，yml配置文件，thrift、protobuf、avro的IDL，dockerfile等等，这些都是DSL。DSL的存在可以大大降低系统的复杂性，当然前提是DSL并不复杂。DSL绝对不能等同于编程语言，但要求近似等同于人类语言。

### 设计DSL的语法结构

在之前开发的系统中，其中一个模块就是一个规则引擎，规则引擎有一个workflow的功能。workflow就是一套事先定义好的flows，执行过程中会因为中间结果的不同而走向不同的flow。在开发workflow的时候，我就设计了一个DSL。

DSL的内容很简单：

```
namespace some.project

init:
ruleSet = [checkAge.groovy, checkCompany.groovy,...]
mode = normal
pass -> blacklist
reject -> return

step blacklist:
ruleSet = [...]
```

首先，是一个namespace的定义；然后必须定义一个init block，init中定义了将要执行的ruleSet，执行ruleSet的时候会根据mode的值选择具体的执行方式，如顺序执行、并行执行、快速失败等等。mode的默认值是normmal，所以mode可以省略。对于ruleSet的执行结果返回三种可能：pass,reject和undefine。对于各个结果可以设定下一步的执行路径，路径可以是直接返回，或是一个step name。默认是直接返回，即return。step可以有0或多个。

这样，我们的DSL的语法结构就出来了。现在我们需要一个parser。

### 什么是Parser？

Parser一段程序，可以从上到下扫一遍DSL文件，parse出DSL的结构信息。Parser的写法就那么几种，原理简单，并不复杂，复杂的地方是，如果我写的DSL出问题了，Parser是否可以及时发现，并在正确的地方抛出异常。

这里我选用了[parboiled](https://github.com/sirthias/parboiled/wiki)来写parser。真的非常好用。parboiled不像antlr那类的生成器，parboiled是一个框架，开发者需要自己用java或scala开发parser。parboiled基于PEG（[Parsing expression grammars](https://en.wikipedia.org/wiki/Parsing_expression_grammar)）。

### 简单解释PEG

PEG还是很好理解的，可以理解为对字符串parse的过程进行拆分并组装。假设我要实现一个四则运算的计算器，就是给一个公式，公式中包含了因子和操作符，操作符包含了加减乘除和左右小括号，因子就是简单的数字。一个公式可能是这样的：1+2，也可能是这样的：1+2\*3+4/2，还可能是这样的：(1+2)*(4-3)。那如何用PEG描述呢？

```
Expr    ← Sum
Sum     ← Product (('+' / '-') Product)*
Product ← Value (('*' / '/') Value)*
Value   ← [0-9]+ / '(' Expr ')'
```

翻译成中文就是：我们要求表达式（Expr）的值，首先求Sum；Sum是一个Product，以及0或多个加／减Product（注意这里的0或多个修饰的是：加／减和product）；而Product是一个Value，以及0个或多个乘除Value；Value或者是由多个0-9的简单数字组成的整数，或者是一个表达式Expr。

能够看出来，这还是一个递归的程序，只不过PEG使用数学公式的方式进行了充分定义。

### Parboiled的优点

尽管Parboiled组织java代码的方式让我难以接受，但是作为DSL的开发工具还是很不错的，与antlr完全不同，可以写java代码，组织AST的方式也十分巧妙。开发者可以使用Var这个接口，嵌套自己的AST，然后在解析到关键变量之后，压到AST中。

回头看我们定义的DSL，这个DSL与前面说的四则运算不同的地方，仅仅是从单行扩充到多行，也就是说换行符是一个tonkenizer。需要考虑block，仅此而已。[代码在这里](https://github.com/magicalne/parboiled-demo/blob/master/src/main/java/PolicyParser1.java)
