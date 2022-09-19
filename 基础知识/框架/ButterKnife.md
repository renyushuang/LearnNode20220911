面试：

第一段 是什么：

1.源码会经过三个处理，源码期、编译器期、运行期

2.所以在编译器期、运行期都可以进行业务代码的处理

ButterKnife通过注解和注解处理器简化了我们findViewById、getResource.getString等

第二段 实现一个注解

1.定义注解

2.定义注解处理器，生成Activity获取其他View对应的Binder类

3.进行注册，即实例化我们新生成的类，将View、String等内容进行绑定

第三段 注解处理器的原理













注解+注解处理器



## ButterKnife

1.ButterKnife初始化

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    // ButterKnife初始化
    ButterKnife.bind(this);
    textView.setText("111");
}
```

2.bind

```java
@NonNull @UiThread
public static Unbinder bind(@NonNull Activity target) {
  View sourceView = target.getWindow().getDecorView();
  return bind(target, sourceView);
}

@NonNull @UiThread
public static Unbinder bind(@NonNull Object target, @NonNull View source) {
  Class<?> targetClass = target.getClass();
  if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
  // target为Activity类对象 ,转至 3.findBindingConstructorForClass
  Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

  if (constructor == null) {
    return Unbinder.EMPTY;
  }

  //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
  try {
    return constructor.newInstance(target, source);
  } catch (IllegalAccessException e) {
    throw new RuntimeException("Unable to invoke " + constructor, e);
  } catch (InstantiationException e) {
    throw new RuntimeException("Unable to invoke " + constructor, e);
  } catch (InvocationTargetException e) {
    Throwable cause = e.getCause();
    if (cause instanceof RuntimeException) {
      throw (RuntimeException) cause;
    }
    if (cause instanceof Error) {
      throw (Error) cause;
    }
    throw new RuntimeException("Unable to create binding instance.", cause);
  }
}
```

3.findBindingConstructorForClass

```java
@Nullable @CheckResult @UiThread
private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
  Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
  if (bindingCtor != null || BINDINGS.containsKey(cls)) {
    if (debug) Log.d(TAG, "HIT: Cached in binding map.");
    return bindingCtor;
  }
  // 获取类的名字，即Activity的名字
  String clsName = cls.getName();
  if (clsName.startsWith("android.") || clsName.startsWith("java.")
      || clsName.startsWith("androidx.")) {
    if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
    return null;
  }
  try {
    // 通过反射，反射Activity_ViewBinding这个类
    Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
    //noinspection unchecked
    bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
    if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
  } catch (ClassNotFoundException e) {
    if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
    bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
  } catch (NoSuchMethodException e) {
    throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
  }
  BINDINGS.put(cls, bindingCtor);
  return bindingCtor;
}
```



通过编译时技术，生成了clsName + "_ViewBinding"的类，来处理findviewByid



## 手写一个ButterKnife

1.创建annotation库，声明注解（注解是java支持的，所以需要在java的项目中去创建注解）

```java
// 类BindView
@Target(ElementType.FIELD) // 声明注解作用 FIELD是成员变量
@Retention(RetentionPolicy.CLASS) // 声明注解的生命周期 源码期、编译期、运行期
public @interface BindView {
    int value();

}

// 类OnClick
@Target(ElementType.METHOD) // 声明注解作用 METHOD是作用在方法
@Retention(RetentionPolicy.CLASS) // 声明注解的生命周期 源码期、编译期、运行期
public @interface OnClick {
    // int []接收多个Click
    int[] value();

}
```

2.创建annotation_compiler，注解处理器

```java
dependencies {
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc4'
    compileOnly 'com.google.auto.service:auto-service:1.0-rc4'

    implementation project(path: ':annotation')
}
```

3.创建注解处理器

```java
@AutoService(Processor.class) // 注册这个类
public class AnnotationCompile extends AbstractProcessor {
 
}
```

2. 必要的声明 支持的版本、处理的注解、打印...

```java
public class AnnotationCompile extends AbstractProcessor {
    Filer filer;// 用来创建文件的

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        filer = processingEnvironment.getFiler();
        printf("111111111111");
//        Messager messager = processingEnvironment.getMessager();

    }

    private void printf(String log) {
        Messager messager = processingEnv.getMessager();
        messager.printMessage(Diagnostic.Kind.NOTE, log);
    }

    // 支持的java版本 也可以通过注解的方式进行设置 @SupportedSourceVersion()
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return processingEnv.getSourceVersion();
    }

    // 声明要注解的内容 也可以通过注解的方式进行设置 @SupportedAnnotationTypes()
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new HashSet<String>();
        types.add(BindView.class.getCanonicalName());
        types.add(OnClick.class.getCanonicalName());
        return types;
    }


}
```

3.process进行注解内容的解析，并创建要使用到的类clsName + "_ViewBinding"

```java
@Override
public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
		
  	// 解析节点信息
		findAndParserTarget
		// 根据节点创建类信息
    brewJava 
     
    return false;
}

  JavaFile brewJava(int sdk, boolean debuggable) {
    TypeSpec bindingConfiguration = createType(sdk, debuggable);
    return JavaFile.builder(bindingClassName.packageName(), bindingConfiguration)
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
  }
	
		// 解析节点信息
    private Map<TypeElement, ElementType> findAndParserTarget(RoundEnvironment roundEnvironment) {
        Map<TypeElement, ElementType> hashMap = new HashMap<>();
        Set<? extends Element> viewElement = roundEnvironment.getElementsAnnotatedWith(BindView.class);

        Set<? extends Element> viewClickElement = roundEnvironment.getElementsAnnotatedWith(OnClick.class);


        // 获取成员变量 的节点，放入到对应的class TypeElement中
        for (Element element : viewElement) {
            VariableElement variableElement = (VariableElement) element;
            // 获取上一个节点，即类节点
            TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();

            ElementType elementType = hashMap.get(typeElement);
            if (elementType == null) {
                elementType = new ElementType();
                hashMap.put(typeElement, elementType);
            }
            List<VariableElement> variableElementList = elementType.getVariableElementList();
            variableElementList.add(variableElement);
        }

        // 获取方法的节点，放入到对应的class TypeElement中
        for (Element element : viewClickElement) {
            ExecutableElement exeElement = (ExecutableElement) element;
            // 获取上一个节点，即类节点
            TypeElement typeElement = (TypeElement) exeElement.getEnclosingElement();

            ElementType elementType = hashMap.get(typeElement);
            if (elementType == null) {
                elementType = new ElementType();
                hashMap.put(typeElement, elementType);
            }
            List<ExecutableElement> executableElements = elementType.getExecutableElements();
            executableElements.add(exeElement);

        }


        return hashMap;
    }
```
