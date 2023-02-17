

# 第四部分 性能优化

## 4.1 bitmap优化

面试题：

1.bitmap是创建在java内存还是native内存? native内存

2.png 1M的和10M大小相同是1080*1920 ARGB8888占用内存大小是否相同？相同



### 4.1.1 bitmap在内存中的创建过程

#### 1.从源码的角度

**java层：**

```java
public final class Bitmap implements Parcelable {
  //c层创建完成会将内存地址返回
  private final long mNativePtr;
  public final class Bitmap implements Parcelable {
        //Bitmap可以通过createBitmap进行创建，当然也有一些其他的方式
        public static Bitmap createBitmap(int width, int height,
              @NonNull Config config, boolean hasAlpha) {
          return createBitmap(null, width, height, config, hasAlpha);
        }

   public static Bitmap createBitmap(@Nullable DisplayMetrics display, int width, int height,@NonNull Config config, boolean hasAlpha, @NonNull ColorSpace colorSpace) {
          ...
          // 经过多种重载的调用后，最终会进入到一个nativeCreate的调用，进入到c层。
          Bitmap bm = nativeCreate(null, 0, width, width, height, config.nativeInt, true,
                  colorSpace == null ? 0 : colorSpace.getNativeInstance());
          ...
          return bm;
      }

```

**C层：**

```c++
/framework/base/core/anndroid/jni/android/graphics/Bitmap.cpp
//c层方法定义
static const JNINativeMethod gBitmapMethods[] = {
    {   "nativeCreate",             "([IIIIIIZ[FLandroid/graphics/ColorSpace$Rgb$TransferParameters;)Landroid/graphics/Bitmap;",
        (void*)Bitmap_creator },
  
  
static jobject Bitmap_creator(JNIEnv* env, jobject, jintArray jColors,
                              jint offset, jint stride, jint width, jint height,
                              jint configHandle, jboolean isMutable,
                              jfloatArray xyzD50, jobject transferParameters) {
    SkBitmap bitmap;
  	// bitmap的内存大小和 width、height、colorType有关
    bitmap.setInfo(SkImageInfo::Make(width, height, colorType, kPremul_SkAlphaType, colorSpace));
		// 进行内存的分配
    sk_sp<Bitmap> nativeBitmap = Bitmap::allocateHeapBitmap(&bitmap, NULL);
    if (!nativeBitmap) {
        return NULL;
    }
    return createBitmap(env, nativeBitmap.release(), getPremulBitmapCreateFlags(isMutable));
}
 
```

#### 2.从内存的角度















### 4.1.2 关于bitmap复用面试题

### 4.1.3 Bitmap底层渲染


###  4.1.4 手写案例实现bitmap渲染gif