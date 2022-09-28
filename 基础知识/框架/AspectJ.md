传统方案：OOP面向对象

​	通过手动埋点的方式进行数据统计

面向切面：AOP

​	只需要标记的方法，得到执行前和执行后的时间



AOP应用场景：权限校验、日志上传、行为统计、性能监测等 





### 耗时统计

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface ExecuteTime {
}
```

```java
// 通过自定义注解，标注要监听的方法
@ExecuteTime
public void getTime() {

    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

```java
/**
 * 声明织入类，声明之后就由Aspectj编译器进行编译
 */
@Aspect
public class TimeAspectj {
    private static final String TAG = "TimeAspectj";

    // 创建切入点
    @Pointcut("execution(@ExecuteTime * * (..))")
    public void getTimes() {


    }

    /**
     * @param joinPoint 用来得到注解中 getTimes()的放法
     */
    @Around("getTimes()")
    public void invokeGetTime(ProceedingJoinPoint joinPoint) {
        long start = System.currentTimeMillis();
        try {
            joinPoint.proceed();

        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        long end = System.currentTimeMillis();
        Log.i(TAG, "getTime duration : " + joinPoint + " --- " + (end - start));
    }

}
```

### 权限校验

这个简单些...

### 埋点上传

1.定义注解 获取方法

2.定义参数 注解 获取参数
