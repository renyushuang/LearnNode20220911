## Groovy类与方法

### getter/setter

- 默认会生成getter、setter方法
- 并且可以直接像使用成员变量的方法来自动判断调用getter/setter
- 当进行赋值时进行调用setter方法
- 当直接访问值时调用的事getter
- 使用.@才是真正的直接访问变量，跳过默认的getter/setter方法调用



### 多种访问get/set方式

通过.变量名来访问变量，实际上还是调用getter/setter方法

```groovy
Car car = new Car()

car.'miles' = 10000

def str = 'miles'

car."$str" = 2000

car['miles'] = 3000
```

- 通过.'变量名'的形式，代码可以变得更动态、灵活



### Groovy中的private不被限制

- 所有变量默认是public的
- 如果要设置为私有禁止访问，仅声明private是不行的。依然可以使用'.'直接访问

- 即时吧这段代码放入到另一个package下面也不行
- 重载这个变量的getter/setter方法，并且调用方法时抛出异常
- java中如果没有显式的制定访问修饰符（public、protected、private）那么默认是包访问权限，Groovy使用@PackageScope



### 构造方法

- 构造方法重载规则跟Java一样
- 但是要注意，如果没有指定具体参数的类型时，默认推断类型时Object
- 在构造方法中传入具名参数，但是要注意：传入的参数都是键值对，实则就是一个Map类型！
- 这种方式传入的参数会自动拆解Map并且调用setter方法对应进行赋值
- 如果参数重还有非键值对的传参，就会把这些键值对当成Map了不再进行自动拆解复制。如果要有对应的构造方法。



### 操作重载符

- 重写每个操作符号对应的方法可以实现操作符重载

```groovy
a+b			a.plus(b)
a-b			a.minus(b)
a*b			a.multiply(b)
...
```



## Groovy闭包

### Groovy闭包

- Groovy中可以理解为闭包就是可执行的代码块，或匿名函数。闭包在使用上与函数与许多共通之处，但是闭包可以作为一个函数的参数。
  - 默认情况下，闭包能接收一个参数 ，且参数字段默认使用it
  - 在箭头前面没有定义参数，这时候闭包不能传入参数
  - 可以定义多个接收的参数
  - 使用参数默认值，跟方法使用规则一样！

```groovy
def closure = {
	println it
}
close(111)

def closure = {
	->// 无参数
	println it
}
def closure = {
	a,b,c->// 多参数
	println it
}

close(111)
```



### 柯里化闭包

- 使用curry方法给闭包参数加默认值，生成的新的闭包，就等于设置了茶壶是默认值的闭包
- // def curriedClosure = closure2.curry("20") // curry从左到右设置默认参数



### 闭包与接口/类进行转换

- 即使定义的方法传入的是闭包，但是如果传入的对象的类型也有call方法，那么，是可以执行这个对象的call方法的，实际上，闭包执行的也是call方法。
- 在一个对象上调用(),表示调用这个对象的call方法：
  - def action = new Action()
  - action()

### 闭包重要的成员

```groovy
private Object delegate;
private Object owner;
private Object thisObject;
private int resolveStrategy = OWNER_FIRST;
protected Class[] parameterTypes;
protected int maximumNumberOfPatameters;
```



### 闭包中的代理策略

- delegate默认就是owner，在执行脚本中可以直接调用方法。
- 但是如果方法放入类中就可以了，可以通过修改代理的方式，让它能够调用。
- 代理策略，默认的是选择owner
- 闭包有以下代理策略：
  - groovy.lang.Closure#DELEGATE_FIREST // delegate优先
  - groovy.lang.Closure#DELEGATE_ONLY // 只在delegate中找
  - groovy.lang.Closure#OWNER_FIREST // owner优先
  - groovy.lang.Closure#OWNER_ONLY // 只在owner中找
  - groovy.lang.Closure#TO_SELF // 只在自身找（闭包内部），意义不大



## 动态特性及元编程

## 动态特性

- 根据上下文推断具体类型
  - def file = new File("");
  - file.geParent()
- CallSite动态调用节点
- java中使用继承才能实现多态，而Groovy轻而易举
  - 优势：灵活
  - 缺点：编译时不会检查类型，运行时报错

### 元编程

- java中可以通过反射，可以在运行时动态的获取类的属性，方法等信息，然后反射调用。但是没法直接做到往内中注入变量、方法；不过java也有动态字节码技术：ASM、JVM TI，javasisist等
- MOP(元对象协议)：Meta Object Protocal
- Groovy直接可以使用MOP进行元编程，我们可以给与应用当前的状态，动态添加或修改变量类中的方法和行为。比如在某个Groovy类中并没有实现某个方法，这方法的具体操作由服器来控制，使用元编程，为这个类动态添加方法，或者替换原来的实现，然后可以进行调整。













