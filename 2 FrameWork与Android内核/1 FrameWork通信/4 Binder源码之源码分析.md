| 进程通信机制                                                 |
| ------------------------------------------------------------ |
| 为什么会有aidl文件，不直接继承会更好吗                       |
| Android为什么不用linux已有的IPC进程通信 (管道,共享内存 Scoket,File) |
| ServiceManager与Binder接口的源码分析                         |
| Binder核心函数mmap源码分析                                   |
| mmap函数为何能做到只需拷贝一次(mmap原理详解)                 |
| 项目应用(手写Binder通信框架)                                 |

## mmap使用 

```c
Java_com_maniu_bindermaniu_ManiuBinder_write(JNIEnv *env, jobject thiz) {
//物理内存 ----活跃代码    ----》  磁盘
//物理内存--- 磁盘
    std::string file = "/sdcard/binder";
    int m_fd = open(file.c_str(), O_RDWR | O_CREAT, S_IRWXU);
//    4k  物理内存映射    4k   文件  4k

    ftruncate(m_fd, 4096);
//    java能 物理内存
//用户空间   能  1 不能2
//用户 不能够直接申请物理内存
//  mmap
//物理地址1    虚拟地址2
    int8_t *m_ptr = static_cast<int8_t *>(mmap(0, 4096,
            PROT_READ | PROT_WRITE, MAP_SHARED, m_fd,
                                               0));
    std::string data("码牛 用代码成就你成为大牛的梦想");
    memcpy(m_ptr, data.data(), data.size());
}
//B  能 1  不能2
extern "C"
JNIEXPORT jstring JNICALL
Java_com_maniu_bindermaniu_ManiuBinder_read(JNIEnv *env, jobject thiz) {
//    怎么把ptr
    std::string file = "/sdcard/binder";
    //打开文件
    int m_fd = open(file.c_str(), O_RDWR | O_CREAT, S_IRWXU);
    ftruncate(m_fd, 4096);
    int8_t *m_ptr = static_cast<int8_t *>(mmap(0, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd,
                                               0));
//申请内存
    char *buf = static_cast<char *>(malloc(100));

    memcpy(buf, m_ptr, 100);
//ptr    读 能 1 不能2
    std::string result(buf);
    __android_log_print(ANDROID_LOG_ERROR, "david", "读取数据:%s", result.c_str());
    //取消映射
    munmap(m_ptr, 4096);
    //关闭文件
    close(m_fd);
    return env->NewStringUTF(result.c_str());
}
```

## 8  AIDL生成java类的细节   

- stub数据接收方

- prox数据发送方
  - 处理这个请求是那个类
  - 方法通过方法的索引
  - 方法携带的参数
  - transcat进行发送-->binder_transcat--->copy_from_user数据拷贝-->copy_from_user数据头拷贝，目标进程、目标线程、验证信息



### ServicesManager

服务注册
服务发现
服务调用



服务注册:ServiceManager.getDefault().add("nname",.class);

服务发现:ServiceManager.getDefault().getInstance(.class);



## AMS启动

```java
//ServicesManager中进行startService
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
```

实际上：通过反射创建对象Lifecycle

```java
Constructor<T> constructor = serviceClass.getConstructor(Context.class);
service = constructor.newInstance(mContext);
```

Lifecycle是AMS的包装类，通过getService获取到AMS

```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }
  
    public ActivityManagerService getService() {
        return mService;
    }
}
```





