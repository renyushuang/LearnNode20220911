- APT补充
  - 策略模式
  - SPI机制分析
  - 通过javac分析apt执行原理
- ASM
  - 逆波兰表达式
  - java文件转class文件基本原则
  - ASM框架完成字节码插桩





### SPI机制

SPI 全名称，Service Provider Interface

是一种服务发现机制，它通过再ClassPath路径下的META-INF/Services文件夹找文件，自动加载文件里定义的类。

```java
//SPI机制
//对象初始化
ServiceLoader<SPIService> load = new ServiceLoader.load(SPIService.class);
Iterator<SPIService> iterator = load.iterator();
//完成方法注入调用
while(iterator.hasNext()){//获取文本文件，名称
  SPIService next = iterator.next()//反射创建对象 
  next.exexute()
}

```

ServecesLoader 发现服务

### javac 源码

Main.java main

compile

另一个Main

compile

JavaComoile.instance

JavaComoile.compile

initProcessAnnnotations--->setProcess--->ServiceIrerator

processAnnotations--->doProcess—rond run---> callProcessor



morTodo





### ASM

