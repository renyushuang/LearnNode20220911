## Gradle介绍

1.基于Apache的Ant和Maven概念的开源自动化构建工具。核心是基于java实现的，可以把gradle堪称一个轻量级的java程序

2.gradle使用groovy、kotlin等芋圆编写自定义脚本，取代了Ant和Maven使用xml配置文件的方式，很大程度简化了开发时对项目的构建要做的配置，使用更加灵活



### 为什么要学习Gradle

Gradle不仅是目前Android最主流的构建工具。而且不少技术领域如组件化、插件化、热修复，及构建系统，都需要通过Gradle来实现，不懂Gralde将无法完成上述事情，所以学习Gradle非常必要



### 关于项目构建

- 对于Java应用程序，编写好的Java代码需要变异成.class文件才能够执行，所以任何的Java应用程序开发，最终都需要经过这一步。
- 编译好了这些class文件，还需要对其进行打包。打包不仅针对这些class文件，还有所有的资源文件等。比如web工程打包成jar或者war就包含了自己的资源文件，然后方法哦服务端运行
- Android工程变好的class问我呢间还要被打包到dex包中，并且所有资源文件进行合并处理，甚至还需要对最终打包出来的文件进行加密和签名处理等等



### Android build 构建出APK

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt5ibH61k2aSyMGBcayEqf6GD69cxYhPvt9V1JoqeXHhiaoqGxU6TictAHCD9UNM7qJ1icw76dBwrICMnw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### Gradle提供了什么

- 对多工程的构建支持非常出色，尤其是工程依赖问题，并支持局部构建
- 多种方式依赖管理：如远程Maven仓库，nexus私服，ivy仓库或者本地文件系统等
- 支持传递性依赖管理
- 轻松迁移项目工程
- 基于Groovy等芋圆构建脚本，简便且灵活
- 免费开源，并且整体设计是以作为一种语言为导向的，而非成为一个严格死板的框架



## Groovy介绍

Groovy是一种基于JVM的面接开发语言，它结合了Python、Ruby和Smalltalk的许多强大特性，Groovy代码能够与Java代码很好的结合，也能用于扩展现有代码。由于其运行再JVM的特性，Groovy也可以使用其他非Java语言编写的库

### Groovy&Java&Kotlin

- Groovy、Java及Kotlin都是基于JVM的开发语言
- Groovy基于Java，在语法上基本相似，但也做了很多自己的扩展。Groovy除了可以面向对象编程，还可以作为纯粹的脚本语言，这一点和Kotlin是一样的
- Groovy和Kotlin都有自己支持的DSL语法，两者有许多的共通之处
  - Groovy： 我不是针对Java，而是所有不能愉快的写DSL的都是垃圾
  - Clojure：我真的是不是针对Java，关于并发，我有一个大胆的想法
  - Scala：我真的是不是针对Java，关于并发，我有一个大胆的想法
  - Kotlin：我就是处处针对Java
  - java退出了群聊

### Groovy特性

- 同时支持静态类型和动态类型
- 支持运算符重载
- 支持DSL语法特性
- 本地语法列表和关联数组
- 各种标记语言，如XML和HTML原生支持
- 对正则表达式的本地支持
- Groovy和Java非常相似，可以无缝衔接使用Java
- 支持使用现有的Java库，并做了一定的扩展

