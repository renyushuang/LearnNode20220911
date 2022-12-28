| xml布局加载与显示原理        |
| ---------------------------- |
| 如何实现监听布局解析过程     |
| 项目应用(手写网易云换肤框架) |
| Resource资源生成的奥秘       |
| Tinker，Sophix资源替换原理   |

## LayoutInflater源码分析

### 第一点 了解XML的解析过程（为了突出第二点）

```java
class AppCompatDelegateImplV9 extends AppCompatDelegateImplBase
        implements MenuBuilder.Callback, LayoutInflater.Factory2 {
        
    @Override
    public void setContentView(int resId) {
      	// 创建DecorView
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
      	// 创建完DecorView，拿到其中的content的FrameLayout作父布局给自定义的xml
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mOriginalWindowCallback.onContentChanged();
    }
```

进行xml的解析

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }

    final XmlResourceParser parser = res.getLayout(resource);
    try {
      	// parser解析器，root根布局，attachToRoot暂时不知道是什么
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```



```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        try {
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
								// 如果是merge那么将会以外层的ViewGroup作为根布局，这样会节省一层布局
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
              	//在xml中找到根视图
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }

                // Inflate all children under temp against its context.
              	// 进行子View的解析,把自定义的XMl所有除根布局之外的控件全部实例化
                rInflateChildren(parser, temp, attrs, true);

                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(parser.getPositionDescription()
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        return result;
    }
}
```



```java
//得到一个View控件将其返回回去
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    try {
      	// 如果Activity继承自AppCompatActivity，那么将会在Factory的进行实例
        // 这里有两个方法，那么这两个Factory，主要是为了扩展，避免在同一个里面进行代码的修改
      	View view;
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }
				// 如果Activity继承普通的Activity，则将会在这里初始化
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
              	// 现在包含的名字是带包名的
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }
				// 根中这个返回的View
        return view;
    } catch (InflateException e) {
        throw e;
    } 
}
```



Factory2.onCreateView的实现

```java
class AppCompatDelegateImplV9 extends AppCompatDelegateImplBase
        implements MenuBuilder.Callback, LayoutInflater.Factory2 {
  
    @Override
    public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        // First let the Activity's Factory try and inflate the view
        final View view = callActivityOnCreateView(parent, name, context, attrs);
        if (view != null) {
            return view;
        }

        // If the Factory didn't handle it, let our createView() method try
        return createView(parent, name, context, attrs);
    }
  
  	    @Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
    		//...

        return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
                true, /* Read read app:theme as a fallback at all times for legacy reasons */
                VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
        );
    }
  
```



```java
class AppCompatViewInflater {
  			// parent 理解为ContentView，name控件的名字，attrs
  			// 用来生成空间，控件名字和控件属性的集合
  			// AppCompat是为了兼容版本
		    public final View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs, boolean inheritContext,
            boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
     
        View view = null;

        // We need to 'inject' our tint aware Views in place of the standard framework versions
        switch (name) {
            case "TextView":
            		// AppCompatTextView 移花接木，也是为了兼容
                view = new AppCompatTextView(context, attrs);
                break;
            case "ImageView":
                view = new AppCompatImageView(context, attrs);
                break;
            case "Button":
                view = new AppCompatButton(context, attrs);
                break;
            case "EditText":
                view = new AppCompatEditText(context, attrs);
                break;
            case "Spinner":
                view = new AppCompatSpinner(context, attrs);
                break;
            case "ImageButton":
                view = new AppCompatImageButton(context, attrs);
                break;
            case "CheckBox":
                view = new AppCompatCheckBox(context, attrs);
                break;
            case "RadioButton":
                view = new AppCompatRadioButton(context, attrs);
                break;
            case "CheckedTextView":
                view = new AppCompatCheckedTextView(context, attrs);
                break;
            case "AutoCompleteTextView":
                view = new AppCompatAutoCompleteTextView(context, attrs);
                break;
            case "MultiAutoCompleteTextView":
                view = new AppCompatMultiAutoCompleteTextView(context, attrs);
                break;
            case "RatingBar":
                view = new AppCompatRatingBar(context, attrs);
                break;
            case "SeekBar":
                view = new AppCompatSeekBar(context, attrs);
                break;
        }

        if (view == null && originalContext != context) {
            // If the original context does not equal our themed context, then we need to manually
            // inflate it using the name so that android:theme takes effect.
            view = createViewFromTag(context, name, attrs);
        }

        if (view != null) {
            // If we have created a view, check its android:onClick
            checkOnClickListener(view, attrs);
        }

        return view;
    }
```

普通的Activity的createView

```java
static final Class<?>[] mConstructorSignature = new Class[] {
            Context.class, AttributeSet.class};

public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
  	// 为什么使用一个容器进行获取z
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
      	// 不为空，并且验证是否合格，合格的标准为类加载器相同
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
          	// 如果这个是TextView的话，那么就会通过类加载器进行加载
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
          	// 通过getConstructor，得到第二个构造方法
            constructor = clazz.getConstructor(mConstructorSignature);
          	// 打开权限
            constructor.setAccessible(true);
          	// 将View进行缓存，避免都通过反射创建，优化
            sConstructorMap.put(name, constructor);
        } else {
          
        }

				//	执行构造方法，并将View进行返回
        final View view = constructor.newInstance(args);
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        mConstructorArgs[0] = lastContext;
        return view;

    } catch (NoSuchMethodException e) {
       
}
```

到这才将根布局给看完，接下来是子View解析

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```



```java
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            throw new InflateException("<merge /> must be the root element");
        } else {
          	// 把当前的节点变成一个对象，然后再添加到viewGroup中
            final View view = createViewFromTag(parent, name, context, attrs);
          	// parent就是外部传进来的容器
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

mFactory是哪里来的来

### 第二点 布局工厂Factory的生成过程以及作用（利用特性进行皮肤更换）

```java
super.onCreate(savedInstanceState);
```

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    final AppCompatDelegate delegate = getDelegate();
    delegate.installViewFactory();
    delegate.onCreate(savedInstanceState);
    if (delegate.applyDayNight() && mThemeId != 0) {
        // If DayNight has been applied, we need to re-apply the theme for
        // the changes to take effect. On API 23+, we should bypass
        // setTheme(), which will no-op if the theme ID is identical to the
        // current theme ID.
        if (Build.VERSION.SDK_INT >= 23) {
            onApplyThemeResource(getTheme(), mThemeId, false);
        } else {
            setTheme(mThemeId);
        }
    }
    super.onCreate(savedInstanceState);
}
```



```java
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
      	// 这个this很重要，
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    } else {
        if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImplV9)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
}
```



```java
public static void setFactory2(
        @NonNull LayoutInflater inflater, @NonNull LayoutInflater.Factory2 factory) {
    IMPL.setFactory2(inflater, factory);
}
```



```java
static class LayoutInflaterCompatBaseImpl {
    @SuppressWarnings("deprecation")
    public void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
        final LayoutInflater.Factory2 factory2 = factory != null
                ? new Factory2Wrapper(factory) : null;
        setFactory2(inflater, factory2);
    }

    public void setFactory2(LayoutInflater inflater, LayoutInflater.Factory2 factory) {
      // 在这里将Factory进行设置
      inflater.setFactory2(factory);

        final LayoutInflater.Factory f = inflater.getFactory();
        if (f instanceof LayoutInflater.Factory2) {
            // The merged factory is now set to getFactory(), but not getFactory2() (pre-v21).
            // We will now try and force set the merged factory to mFactory2
            forceSetFactory2(inflater, (LayoutInflater.Factory2) f);
        } else {
            // Else, we will force set the original wrapped Factory2
            forceSetFactory2(inflater, factory);
        }
    }

    @SuppressWarnings("deprecation")
    public LayoutInflaterFactory getFactory(LayoutInflater inflater) {
        LayoutInflater.Factory factory = inflater.getFactory();
        if (factory instanceof Factory2Wrapper) {
            return ((Factory2Wrapper) factory).mDelegateFactory;
        }
        return null;
    }
}
```

## AppCompatActivity与Factory2

## 换肤框架的实现思路分析

Android中的动态加载技术

动态调用外部的dex，也可以动态加载so库，动态加载dex/jar/apk文件等





## 打造动态主题换肤框架



 

