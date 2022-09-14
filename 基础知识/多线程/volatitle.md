# 作用

防止指令重排序

实现可见性（（在各个线程的工作内存中，volatile变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在一致性问题），但是Java里面的运算并非原子操作，导致volatile变量的运算在并发下一样是不安全的。

不保证原子性



# 实例

```java
public class SingleTon {
    private volatile static SingleTon mInstance;

    private SingleTon() {

    }

    public static SingleTon getInstance() {
        if (mInstance == null) {
            synchronized (SingleTon.class) {
                if (mInstance == null) {
                  	// 创建对象需要三个指令 newInstance、分配内存、赋值，其中 分配内存和赋值是可以进行重排的
                  	// 如果没有volatile修饰，则在第一个mInstance == null中会出现运行时异常
                    mInstance = new SingleTon();
                }
            }
        }
        return mInstance;
    }
}
```

```
0x01a3de0f：mov$0x3375cdb0，%esi；……beb0cd75 33 ；{oop（'Singleton'）} 
0x01a3de14：mov%eax，0x150（%esi）；……89865001 0000 
0x01a3de1a：shr$0x9，%esi；……c1ee09 0x01a3de1d：movb$0x0，
0x1104800（%esi）；……c6860048 100100 
0x01a3de24：lock addl$0x0，（%esp）；……f0830424 00 ；
*putstatic instance ；- Singleton：
getInstance@24

```

通过对比就会发现，关键变化在于有volatile修饰的变量，赋值后（前面mov%eax，0x150（%esi）这句便是赋值操作）多执行了一个“**lock addl ＄0x0**，（%esp）”操作，这个操作相当于一个**内存屏障**（Memory Barrier或Memory Fence，指重排序时不能把后面的指令重排序到内存屏障之前的位置），只有一个CPU访问内存时，并不需要内存屏障；但如果有两个或更多CPU访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性了。

# CAS

是一个原子性的指令，JDK通过CPU的`cmpxchgl`指令的支持

cas中使用了三个基本操作数

- 内存地址V
- 旧的预期地址A
- 要修改的新值B

![](https://upload-images.jianshu.io/upload_images/26777047-54804173d3a33429.gif?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

