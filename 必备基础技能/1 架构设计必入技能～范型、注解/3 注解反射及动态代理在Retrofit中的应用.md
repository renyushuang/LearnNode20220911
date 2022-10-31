# 一、注解

## 1.1 注解的作用和意义是什么

注解+APT 用于生成一些java文件 butterknife、dagger2、hilt、databinding

注解+代码埋点 Aspectj、arouter

注解+反射+动态代理 xutils、Lifecycle

| **级别** | **技术**   | **使用场景**                                                 |
| :------: | ---------- | ------------------------------------------------------------ |
|   源码   | APT        | 在编译期能够获取注解与注解声明的类包括类中所有成员信息，一般用于生成额外的辅助类。 |
|  字节码  | 字节码增强 | 在编译出Class后，通过修改Class数据以实现修改代码逻辑目的。对于是否需要修改的区分或者修改为不同逻辑的判断可以使用注解。 |
|  运行时  | 反射       | 在程序运行期间，通过反射技术动态获取注解与其元素，从而完成不同的逻辑判定。 |

## 1.2 注解的分类：元注解、内置注解、自定义注解

### 元注解：用来对注解类型进行注解的注解类

```java
// 声明作用域
@Target(ElementType.METHOD)
    //ElementType.FIELD, 全局变量
    //ElementType.METHOD, 方法
    //ElementType.PARAMETER,参数

//声明生命周期
@Retention(RetentionPolicy.RUNTIME)
    //RetentionPolicy.SOURCE,源码期
    //RetentionPolicy.CLASS,编译期
    //RetentionPolicy.RUNTIME;运行期

@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value(); 
    int id() default 0;//设置默认值
}
```

# Retrofit

Retrofit-建造者模式-》create （实现动态代理）-》 getPersonInfo
-》（注解解析，url拼接）-》ca1I.enqueue（丢给okhttp，RealCalt切到子结程）-》返口数据enqueue(由子线程切回主线桯）



门面模式 一个类把所有内容都展示出来了



构建者模式 构建 （信息初始化）

	- OkHttpClient
	
	- defaultCallbackExecutor ---> Android -----> handler.post(r); 解决了异步请求，主线程返回的问题
	- List<CallAdapter.Factory> 执行请求的类
	- List<Converter.Factory> 数据格式转换

注解的解析

```java
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, Object... args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          // method就是我们自己创建的接口的method，有注解，有参数
          ServiceMethod serviceMethod = loadServiceMethod(method);
          OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```



getCall时执行动态代理

执行  enqueue 交给Okhttp进行请求



动态代理和静态代理区别

静态代理 

1.自己的类（GamePlayer）代理类（GamePlayerProx）

2.GamePlayerProx包含了GamePlayer想要的功能相当于实现了相同的接口IGamePlayer

3.将自己的信息（new GamePlayer） 传递给代理GamePlayerProx

4.由代理完成想要完成的工作

动态代理（解决掉自己和动态代理类的直接联系）

```java
public class DynamicProxy implements InvocationHandler {
    private Object object;

    public DynamicProxy(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        method.invoke(object, args);
        return object;
    }
}
```

```java
// 和中介公司产生关联 ,因为要代理所以返回的是代理的接口
ProxyInterface p = (ProxyInterface) Proxy.newProxyInstance(andy.getClass().getClassLoader(), andy.getClass().getInterfaces(), new DynamicProxy(andy));
```



**Retrofit什么时候切回的主线程**

enqueue请求成功后，使用的defaultCallbackExecutor 切回的主线程

