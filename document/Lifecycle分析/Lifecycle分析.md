##**Android x中Lifecycle分析**


**1. FragmentActivity的Liecycle**:

```java
   /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @Override
    public Lifecycle getLifecycle() {
        return super.getLifecycle();
    }
```
从上可以知道，FragmentActivity中的`getLifecycle()`会调用父类中的`getLifecycle()`从而获得Lifecycle对象。

查看FragmentActivity中全局代码，发现继承的父类是ComponentActivity。


**2. ComponentActivity中Lifecycle对象**：

查看ComponetActivity，发现ComponentActivity实现了LifecycleeOwner，且进一步通过LifecycleRegistry对象（Lifecyle的子类）来管理。

```java
public class ComponentActivity extends Activity
        implements LifecycleOwner, KeyEventDispatcher.Component {
        
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    
    @Override
    @SuppressWarnings("RestrictedApi")
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 绑定一个空白的Fragment，用于绑定Activiy的生命周期
        ReportFragment.injectIfNeededIn(this);
    }
    
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```
先来看下，LifecycleeOwner接口: 该接口比较简单，用于获取Lifecycle对象。

```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```
接下来，看下ReportFramgent监听绑定Activity的生命周期时，如何进一步将这些生命周期传递给LifecycleResgistery对象。

**3. ReportFragment 监听Activity的生命周期**

查看ReportFramgent发现，会在相应的生命周期(`onActivityCreated()...onDestroy()`)内传递相应的事件给LifecycleRegistry。

```java
public class ReportFragment extends Fragment {

    // 当绑定Acitivity是FragmentActivity，会在其onCreate()中绑定，用于记录FragmentActivity中生命周期。
   // 官方考虑到扩展性，因此在ProcessLifecycleOwner中监听Activity的创建。只要是Activity创建成功，都会默认绑定ReportFramgent，用于记录生命周期
    public static void injectIfNeededIn(Activity activity) {
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }
    
    
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        // 传递Activity中的onCreate()事件。
        dispatch(Lifecycle.Event.ON_CREATE);
    }
    
    //..... 省略其他的生命期回调。
    
    
   // 传递给Liecycle对象，Activity的各种生命周期
   private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                // 将生命周期的事件传递给LifecycleRegistry 。
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

}
```
接下来，查看LifecycleRegistry是如何实现处理Activity的生命周期事件的。

**4. LifecycleRegistry处理相应的生命周期事件**

查看LifecycleRegistry可知，会记录LifecycleOwner的当前状态state，且将LifecycleObserver监听器一一对应的存储在Map中。
 
```
public class LifecycleRegistry extends Lifecycle {
    //用于保存观察者对应一些列的监听者
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
    // 当前的State
    private State mState;
    
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        // 开始传递Activity的生命周期事件
        moveToState(next);
    }

}
```
先来看下如何添加LifecycleObserver，了解如何解析其定义的生命周期的方法。

**4.1 将LifecycleObserver添加到 LifecycleRegistry中**:

查看addObserver()，进一步探究

```java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        //解析，记录LifecycleObserver中定义的方法
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        // 将LifecycleObserver信息的实体与LifecycleObserver一一对应的存储
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        //....省略部分代码
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            //同步一些事件,若是在onResume添加LifecycleObserver，仍然会收到ON_CREATE 事件.
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
```

接下来，查看ObserverWithState是如何解析LifecycleObserver的：

```
    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }
        //... 省略部分代码
    }
```
接下来，查看Lifecycling如何 生成对应的 GenericLifecycleObserver对象：

```
   @NonNull
    static GenericLifecycleObserver getCallback(Object object) {
       //... 省略部分代码，处理一些特殊的LifecycleObserver情况(FullLifecycleObserver、GenericLifecycleObserver)等
        return new ReflectiveGenericLifecycleObserver(object);
    }
```
接下来，查看ReflectiveGenericLifecycleObserver的构造器：

```
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        // 存储LifecycleObserver相应的信息。
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }
}
```

接下来，查看ClassesInfoCache是如何解析：将生命周期的方法解析进行存储，方便后续反射调用。

```
   CallbackInfo getInfo(Class klass) {
        // 先从缓存中获取
        CallbackInfo existing = mCallbackMap.get(klass);
        if (existing != null) {
            return existing;
        }
        //若是没有，则解析。
        existing = createInfo(klass, null);
        return existing;
    }

    private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {
        // 从父类中查找是否有定义生命周期的方法
        Class superclass = klass.getSuperclass();
        Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
        if (superclass != null) {
            CallbackInfo superInfo = getInfo(superclass);
            if (superInfo != null) {
                handlerToEvent.putAll(superInfo.mHandlerToEvent);
            }
        }
        // 从实现接口中，查找是否有定义生命周期的方法
        Class[] interfaces = klass.getInterfaces();
        for (Class intrfc : interfaces) {
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                    intrfc).mHandlerToEvent.entrySet()) {
                verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
            }
        }
        // 查找到当前类中定义生命周期的方法
        Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
        boolean hasLifecycleMethods = false;
        for (Method method : methods) {
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            if (annotation == null) {
                continue;
            }
            hasLifecycleMethods = true;
            Class<?>[] params = method.getParameterTypes();
            int callType = CALL_TYPE_NO_ARG;
            if (params.length > 0) {
                callType = CALL_TYPE_PROVIDER;
                if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. Must be one and instanceof LifecycleOwner");
                }
            }
            Lifecycle.Event event = annotation.value();
            // 检查事件的合法性
            if (params.length > 1) {
                callType = CALL_TYPE_PROVIDER_WITH_EVENT;
                if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. second arg must be an event");
                }
                if (event != Lifecycle.Event.ON_ANY) {
                    throw new IllegalArgumentException(
                            "Second arg is supported only for ON_ANY value");
                }
            }
            if (params.length > 2) {
                throw new IllegalArgumentException("cannot have more than 2 params");
            }
            //构建对象储存生命周期的方法
            MethodReference methodReference = new MethodReference(callType, method);
            verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
        }
        // 将解析好的生命周期方法的map存放在实体类中。
        CallbackInfo info = new CallbackInfo(handlerToEvent);
        // 将解析好后的信息存储
        mCallbackMap.put(klass, info);
        mHasLifecycleMethods.put(klass, hasLifecycleMethods);
        return info;
    }
```


**4.2 LifecycleRegistry将生命周期事件，回调到LifecycleObserver中**：

接下来，handleLifecycleEvent() ,进过一些列的状态State检查,同步操作，最后来到forwardPass()中：

```java
  private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                //开始调度生命周期对应的事件
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }
```
接下来，来到ObserverWithState中：ObserverWithState中的mLifecycleObserver是ReflectiveGenericLifecycleObserver。

```
    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```

接下来，查看ReflectiveGenericLifecycleObserver的onStateChanged() ：

```
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;
    
    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        //当事件传递发生改变，反射调用相应的监听器，进行通知。
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

接下来，CallbackInfo中的invokeCallbacks()中,将相应事件对应的MethodReference 。
MethodReference通过invokeCallback(),反射调用LifecycleObserver对应的方法。

```java
class ClassesInfoCache {

     static class CallbackInfo {
        @SuppressWarnings("ConstantConditions")
        void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
            invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
            invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                    target);
        }
        private static void invokeMethodsForEvent(List<MethodReference> handlers,
                LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
            if (handlers != null) {
                for (int i = handlers.size() - 1; i >= 0; i--) {
                    handlers.get(i).invokeCallback(source, event, mWrapped);
                }
            }
        }
    }

    static class MethodReference {
        final int mCallType;
        final Method mMethod;
        
        // 最后反射调用LifecycleObserver中的定义事件的方法。
        void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
            //noinspection TryWithIdenticalCatches
            try {
                switch (mCallType) {
                    case CALL_TYPE_NO_ARG:
                        mMethod.invoke(target);
                        break;
                    case CALL_TYPE_PROVIDER:
                        mMethod.invoke(target, source);
                        break;
                    case CALL_TYPE_PROVIDER_WITH_EVENT:
                        mMethod.invoke(target, source, event);
                        break;
                }
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Failed to call observer method", e.getCause());
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        }
}
```



##**App进程中Lifecycle模块的初始化和Activity的监听**

在Studio中全局检索ProcessLifecycleOwner，查看其源码。


**1. ProcessLifecycleOwner类**

```
public class ProcessLifecycleOwner implements LifecycleOwner {

    private ActivityInitializationListener mInitializationListener =
            new ActivityInitializationListener() {
                @Override
                public void onCreate() {
                }

                @Override
                public void onStart() {
                    activityStarted();
                }

                @Override
                public void onResume() {
                    activityResumed();
                }
            };
            
      void attach(Context context) {
        mHandler = new Handler();
        mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            
                // 当每个FragmentActivity创建时候，会设置为其一一对应的ReportFragment设置监听器
                ReportFragment.get(activity).setProcessListener(mInitializationListener);
            }

            @Override
            public void onActivityPaused(Activity activity) {
                activityPaused();
            }

            @Override
            public void onActivityStopped(Activity activity) {
                activityStopped();
            }
        });
    }              
}
```

**2. ProcessLifecycleOwnerInitializer类** 

进一步检索，发现在ProcessLifecycleOwnerInitializer中，初始化ProcessLifecycleOwner类。

ProcessLifecycleOwnerInitializer是一个ContentProvider子类。

在app进程启动时，创建Application后的attachBaseContext()之后会安装ContentProvider。因此，也会进行Lifecycle相关模块的初始化。

```java
public class ProcessLifecycleOwnerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        LifecycleDispatcher.init(getContext());
        ProcessLifecycleOwner.init(getContext());
        return true;
    }
    //..... 省略部分代码
}
```
**3. LifecycleDispatcher 类**

LifecycleDispatcher为App进程内每个Activity都添加一个对应的ReportFragment对象，用于监听Activity的生命周期。

实际上FragmentActivity已经添加过ReportFragment，但实际上，并不是全部的开发都会继承FragmentActivity。因此，为了Activity中正常使用(实现LifecycleOwner接口)，也需要去添加。

```
class LifecycleDispatcher {

    private static AtomicBoolean sInitialized = new AtomicBoolean(false);

    static void init(Context context) {
        if (sInitialized.getAndSet(true)) {
            return;
        }
        ((Application) context.getApplicationContext())
                .registerActivityLifecycleCallbacks(new DispatcherActivityCallback());
    }
    @SuppressWarnings("WeakerAccess")
    @VisibleForTesting
    static class DispatcherActivityCallback extends EmptyActivityLifecycleCallbacks {

        @Override
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            //为每个activity都添加一个对应的ReportFragment对象。
            ReportFragment.injectIfNeededIn(activity);
        }

        @Override
        public void onActivityStopped(Activity activity) {
        }

        @Override
        public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }
    }
}
```


