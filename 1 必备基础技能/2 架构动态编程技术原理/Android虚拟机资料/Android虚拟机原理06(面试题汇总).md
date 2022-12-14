##### 面试题一

**1.1 虚拟机是什么   JVM 与 Art   Dalvik 之间区别？**

JVM虚拟机与Android虚拟机区别

1. Android虚拟机执行的是.dex格式文件 jvm执行的是.class文件
2. class文件存在很多的冗余信息，dex工具会去除冗余信息
3. Android虚拟机是基于寄存器的虚拟机 而jvm执行是基于虚拟栈的虚拟机

1. 1.2 Art虚拟机与Dalvik虚拟机区别

> 1. Dalvik下，应用每次运行都需要通过即时编译器（JIT）将字节码转换为机器码，即每次都要编译加运行，这虽然会使安装过程比较快，但是会拖慢应用以后每次启动的效率。
>
>    
>
> 2. 而在ART 环境中，应用在第一次安装的时候，字节码就会预编译（AOT）成机器码，这样的话，虽然设备和应用的首次启动（安装慢了）会变慢，但是以后每次启动执行的时候，都可以直接运行，因此运行效率会提高。
>
>    
>
> 3. ART占用空间比Dalvik大（字节码变为机器码之后，可能会增加10%-20%），这也是著名的“空间换时间大法"。
>
>    
>
> 4. Art预编译也可以明显改善电池续航，因为应用程序每次运行时不用重复编译了，从而减少了 CPU 的使用频率，降低了能耗。 

##### 面试题二

**那dex和class到底在结构上有什么区别呢  (又或者为什么Android虚拟机一定要用dex呢，而不用class)**

**心理分析:** 面试官想考你对  dex文件编码结构了不了解。从侧面反映你是否做过热修复，dex加固方面反编译 的技术，这种技术一定对他们目前是非常重要的

总结

> 1. dex文件减少整体的文件尺寸  dex更像是一种压缩文件，一次可以表示更多的class。class像是一种单个文件
> 2. Android虚拟机加载类时 只对dex需要一次IO可以加载很多新类，而class需要加载多次IO，Android虚拟机提高查找速度
> 3. dex指令更加密集。class指令比较多
> 4. dex 寄存器设计方便寻址，class java栈需要更多次load与store指令
> 5. dex适合于移动设备，性能不太高的要求。class适合PC大内存，单指令小的情况下可以快速执行

 、

##### 面试题三

**1.4.1 Android虚拟机中寄存器起什么作用，与栈的区别在哪里（又或者基于栈与基于寄存器的架构，谁更快？）**

面试心里分析

看看现在的实际处理器，大多都是基于寄存器的架构，从侧面反映出它比基于栈的架构更优秀。

一般认为基于寄存器的架构对VM来说也是更快的，

**总结**

**原因是：**

> 1. 虽然没有地址(无变量声明)指令更紧凑，但完成操作需要更多的load/store指令，也意味着更多的指令分派（instruction dispatch）次数与内存访问次数；
> 2. 访问内存是执行速度的一个重要瓶颈，二地址或三地址指令虽然每条指令占的空间较多，但总体来说可以用更少的指令完成操作，指令分派与内存访问次数都较少。



##### 面试题四: 

##### 虚拟机连环炮系列 Arm指令究竟是什么指令，能说说他与字节码指令的区别吗**

面试心里分析

总结

> 字节码指令  和  Arm指令内容是不一样
>
> 如 同样一个  a+b
>
> 在 jvm的指令   iadd   idiv   imul   
>
> 但是在dalvik指令是   add-int  mul-int
>
> arm指令是由arm公司开发的。 指令含有地址，而字节码指令没有地址
>
> 字节码指令是 sun公司开发，简单高效

 

##### 面试题五

**为什么Art虚拟机比Dalvik虚拟机运行速度高**

（1）在Dalvik下，应用每次运行都需要通过即时编译器（JIT）将字节码转换为机器码，即每次都要编译加运行，这虽然会使安装过程比较快，但是会拖慢应用以后每次启动的效率。而在ART 环境中，应用在第一次安装的时候，字节码就会预编译（AOT）成机器码，这样的话，虽然设备和应用的首次启动（安装慢了）会变慢，但是以后每次启动执行的时候，都可以直接运行，因此运行效率会提高。

（2）ART占用空间比Dalvik大（字节码变为机器码之后，可能会增加10%-20%），这也是著名的“空间换时间大法"。

（3）预编译也可以明显改善电池续航，因为应用程序每次运行时不用重复编译了，从而减少了 CPU 的使用频率，降低了能耗。

表现结果

1. 系统性能的显著提升
2. 应用启动更快、运行更快、体验更流畅、触感反馈更及时
3. 更长的电池续航能力
4. 支持更低的硬件