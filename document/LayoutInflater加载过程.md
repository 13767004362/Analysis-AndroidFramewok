

第一种加载xml的布局方式：
```
  View.inflate();
```
第二种加载xml的布局方式：
```
 LayoutInflater.from().inflate()；
```
查看一下View.inflate()中的代码：
```java
public static View inflate(Context context, @LayoutRes int resource, ViewGroup root) {
        LayoutInflater factory = LayoutInflater.from(context);
        return factory.inflate(resource, root);
}
```
从上可知,View.inflate()也是内部调用LayoutInflater.inFlate()。

接下来,了解LayoutInflater是如何创建,XmlResourceParse如何解析标签,如何ClassLoader生成View.



### **1. 获取LayoutInflater对象**



看下LayoutInflater.from(),如何获取到LayoutInflater对象的。

```java
   public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```


[ContextImpl](https://www.androidos.net.cn/android/7.0.0_r31/xref/frameworks/base/core/java/android/app/ContextImpl.java)中getSystemService():
```
   @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
```

[SystemServiceRegistry](https://www.androidos.net.cn/android/7.0.0_r31/xref/frameworks/base/core/java/android/app/SystemServiceRegistry.java)中看下,是如何注册的LayoutInflater。
```java
final class SystemServiceRegistry {
  private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();
  static{
         //.... 
         registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
  }
    
}
```
从上可以知道,通过Context获取到LayoutInflater是PhoneLayoutInflater对象。

### **2. 获取XmlResourceParser对象**

[LayoutInflater](https://www.androidos.net.cn/android/7.0.0_r31/xref/frameworks/base/core/java/android/view/LayoutInflater.java)的inflate():
```java
 public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
}
```
重载的inflate():
```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        //生成布局layout对应的xml解析器
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```

查看下[Resources](https://www.androidos.net.cn/android/7.0.0_r31/xref/frameworks/base/core/java/android/content/res/Resources.java)的getLayout():
```java
public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
        return loadXmlResourceParser(id, "layout");
}
```
loadXmlResourceParser():
```java
 XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
            throws NotFoundException {
        final TypedValue value = obtainTempTypedValue();
        try {
            final ResourcesImpl impl = mResourcesImpl;
            impl.getValue(id, value, true);
            if (value.type == TypedValue.TYPE_STRING) {
                return impl.loadXmlResourceParser(value.string.toString(), id,
                        value.assetCookie, type);
            }
            throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                    + " type #0x" + Integer.toHexString(value.type) + " is not valid");
        } finally {
            releaseTempTypedValue(value);
        }
    }
```
[ResourcesImpl](https://www.androidos.net.cn/android/7.0.0_r31/xref/frameworks/base/core/java/android/content/res/ResourcesImpl.java)的loadXmlResourceParser():

```java
  XmlResourceParser loadXmlResourceParser(@NonNull String file, @AnyRes int id, int assetCookie,
            @NonNull String type)
            throws NotFoundException {
        if (id != 0) {
            try {
                synchronized (mCachedXmlBlocks) {
                    final int[] cachedXmlBlockCookies = mCachedXmlBlockCookies;
                    final String[] cachedXmlBlockFiles = mCachedXmlBlockFiles;
                    final XmlBlock[] cachedXmlBlocks = mCachedXmlBlocks;
                    // 先从缓存中获取对应的XmlBlock,再创建对应的XmlResourceParser对象
                    final int num = cachedXmlBlockFiles.length;
                    for (int i = 0; i < num; i++) {
                        if (cachedXmlBlockCookies[i] == assetCookie && cachedXmlBlockFiles[i] != null
                                && cachedXmlBlockFiles[i].equals(file)) {
                            return cachedXmlBlocks[i].newParser();
                        }
                    }
                   // 从AssetsManager中获取到XmlBolock对象
                    final XmlBlock block = mAssets.openXmlBlockAsset(assetCookie, file);
                    if (block != null) {
                        final int pos = (mLastCachedXmlBlockIndex + 1) % num;
                        mLastCachedXmlBlockIndex = pos;
                        final XmlBlock oldBlock = cachedXmlBlocks[pos];
                        if (oldBlock != null) {
                            oldBlock.close();
                        }
                        cachedXmlBlockCookies[pos] = assetCookie;
                        cachedXmlBlockFiles[pos] = file;
                        cachedXmlBlocks[pos] = block;
                        return block.newParser();
                    }
                }
            } catch (Exception e) {
                throw rnf;
            }
        }

    }
```
接下来AssetManager会从apk中根据文件名和xml文件的资源缓存,生成一个XmlBolock的包装类，在生成 XmlResourceParser对象。

### **3. 递归解析xml中每一层的标签,Classloader生成对应的View对象**


接下来，继续查看inflate():

```java
 public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;
            try {
                // xml的检查操作
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
                final String name = parser.getName();
                //布局xml,最外层为mergc的情况
                if (TAG_MERGE.equals(name)) {
                    //root不能为空,且attachToRoot必须为true
                    if (root == null || !attachToRoot) {
                       //因为merge的xml并不代表某个具体的view，只是将它包起来的其他xml的内容加到某个上层ViewGroup中
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    // 
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // 解析出xml中最外层的root view
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        //创建xml对应root view的LayoutParams
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            //若是attach 为false，则为xml中root View设置LayoutParams
                            temp.setLayoutParams(params);
                        }
                    }
                    //递归 解析xml中children控件
                    rInflateChildren(parser, temp, attrs, true);
                    // 若是attach为true，则需要将xml中的布局加载到root中。
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
                    // 若是root为空或者attach为false，则返回xml中的root view。
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
            // 返回 第二个参数root或者xml布局中的root view。
            return result;
        }
    }
```

先来看下xml最层为mergc的情况，看下如何解析。

rInflate():
```java

 void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
            
        final int depth = parser.getDepth();
        int type;
        // 循环递归解析
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }
            final String name = parser.getName();
            if (TAG_REQUEST_FOCUS.equals(name)) {
                //解析requestFocus标签
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                //解析tag的标签
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                //include不能为xml中最外层标签，不然会报错
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                // 解析include的标签
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                // merge只能作为xml中最外层的标签，不然会报错
                throw new InflateException("<merge /> must be the root element");
            } else {
                // 解析出当前这层的view
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                // 递归调用解析xml中下一层中child控件
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }
        //是否完成解析
        if (finishInflate) {
            parent.onFinishInflate();
        }        
}
```
rInflateChildren()实际上也是递归方式，调用rInflate()在解析。


**接下来看下，是如何将xml中view标签解析成对应的view控件**。

createViewFromTag():
```java
 private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
        return createViewFromTag(parent, name, context, attrs, false);
}

```
继续看下,重载的createViewFromTag():
```java
 View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
        
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        //设置相关的theme主题
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }
        // bink标签，是一个特殊的FrameLayout,间隔500毫秒闪烁刷新。
        if (name.equals(TAG_1995)) {
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
            // 先用mFactory2解析,开发者可以自行配置,用于各种用途
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
            // 默认情况下,没有facotry,会通过onCreateView()方法对内置View进行解析，createView()方法进行自定义View的解析。
            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    // 写xml时候,系统自带控件不需要.，例如<TextView/>,因此这里解析内置的view
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        // 解析自定义的view
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
            throw e;

        } catch (ClassNotFoundException e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name, e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;

        } catch (Exception e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name, e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        }
    }
```

PhoneLayoutInflater是LayoutInflater的子类对象,也是一开始的LayoutInflater对象。

来看下[PhoneLayoutInflater](https://www.androidos.net.cn/android/7.0.0_r31/xref/frameworks/base/core/java/com/android/internal/policy/PhoneLayoutInflater.java)中ononCreateView():

```java
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };
    @Override 
    protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        // 循环三种前缀,去创建对应的view
        for (String prefix : sClassPrefixList) {
            try {
                // 通过createView()生成对应的view。
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }
        //若是前三种前缀匹配不上,则父类的onCreateView()中指定的"android.view."去匹配。
        return super.onCreateView(name, attrs);
    }
    
}
```
继续查看,LayoutInflater中的createView():

```java
 public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        // 根据类名获取到对应构造器对象
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        //检查构造器对象是否当前ClassLoader以及父ClassLoader创建的。
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;
        try {
            if (constructor == null) {
                // 若构造器对象为空,则通过ClassLoader根据类名加载出Class对象
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                //缓存起来。
                sConstructorMap.put(name, constructor);
            } else {
                // 缓存的构造器对象
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
                        
                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }

            Object[] args = mConstructorArgs;
            args[1] = attrs;
            //创建对应的View对象
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                //ViewStub是延迟加载,为它设置context.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;

        } catch (Exception e) {
             // ....
        } 
}
```





