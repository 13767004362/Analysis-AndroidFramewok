## **android 创建窗口的过程**

**窗口相关的概念**：

- **窗口**：窗口是接受用户信息的最小单元，即屏幕上的某个独立的界面，可以是Activity界面或者一个对话框Dialog等等。从WindowManagerService角度来看，添加一个窗口实际上是添加一个View对象，即调用WindowManager类的addView()方法。至于这个View对象是来源于Activity还是某个自定义的View都不重要。当WindowManagerService接受到用户信息后，会判断这个消息属于哪个窗口，即哪个View对象，然后通过Binder把这个消息传递给客户端的ViewRoot.W子类。
- **Window类**: 是一个抽象类，该类抽象了客户端窗口的基本行为操作，且定义了一组Callback接口。Activity便是通过实现这个CallBack接口，来获取消息处理的机会的，因消息最初是由WindowManagerService传递给View对象的。
- **ViewRoot类**：客户端创建窗口时，用于与WindowManagerService交互，实现一些逻辑操作。WindowManagerService管理的每个窗口都对应着一个ViewRoot类。
- **W类**：是ViewRoot类中的一个内部类，继承IWindow(Binder类)，用于客户端与WindowManagerService系统服务之间通讯，从而让WindowManagerService能够控制客户端的窗口。

WindowManagerService在这里简称WMS.

创建window的过程：

ActivityThread创建Activity过程：

```java
 private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
 
     //在Activity创建之前，初始化WMS的Binder代理类。
      WindowManagerGlobal.initialize();
      // 创建Activity对象
     Activity a = performLaunchActivity(r, customIntent);

}
```

先来看下，WindowManagerGlobal类的initialize():

先初始化，获取到WMS的IWindowManager代理类，用于WMS系统服务通讯。

```java 
public final class WindowManagerGlobal {

  private static IWindowManager sWindowManagerService;
  public static void initialize() {
        getWindowManagerService();
  }
  public static IWindowManager getWindowManagerService() {
         synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                sWindowManagerService = IWindowManager.Stub.asInterface(
                       ServiceManager.getService("window"));
                try {
                    sWindowManagerService = getWindowManagerService();
                    ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
               }
            }
            return sWindowManagerService;
        }
    }
}
```


接下来,查看Window的setWindowManager():

若是windowManager对象为空，则从全局的window。接着为每个window创建一一对应的WindowManagerImpl。

```
  public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
           boolean hardwareAccelerated) {
           mAppToken = appToken;
           mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
       
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        //会为Window创建一一对应的WindowManagerImpl
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
  }
```

经过ActivityManagerService一系列检查操作后，回到ActivityThread中,执行Activity的onResume()。

ActivityThread的handleResumeActivity():

```
    final void handleResumeActivity(IBinder token,
           boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {

     ActivityClientRecord r = mActivities.get(token);
     
     // 执行Activity的onResume（）
     r = performResumeActivity(token, clearHide, reason);
     
     //开始显示Activity中布局
     f (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                // 设置不可见，但占据位置
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                   // the decor view we have to notify the view root 
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    // 将Activity对应的窗口View对象添加到WindowManager中。
                    wm.addView(decor, l);
               }
           }
}
```




接下来，查看Activity的makeVisible()：

```java 
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            // 开始通知WMS,要显示的信息,要显示Activity对应的DecorView对象 。
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        // 设置decorView对用户可见。
        mDecor.setVisibility(View.VISIBLE);
}
```

接下来，查看 [WindowManagerImpl](http://androidxref.com/7.0.0_r1/xref/frameworks/base/core/java/android/view/WindowManagerImpl.java)的addView():

```java

90    @Override
91    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
92        applyDefaultToken(params);
93        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
94    }
```

接下来，查看：[WindowManagerGlobal](http://androidxref.com/7.0.0_r1/xref/frameworks/base/core/java/android/view/WindowManagerGlobal.java)的addView():

```
   public void addView(View view, ViewGroup.LayoutParams params,
           Display display, Window parentWindow) {
           
           //... 检查参数
           ViewRootImpl root;
           View panelParentView = null;
           synchronized (mLock) {
           
                // 创建ViewRootImpl对象
                root = new ViewRootImpl(view.getContext(), display);
                view.setLayoutParams(wparams);
                mViews.add(view);
                mRoots.add(root);
                mParams.add(wparams);
           }
        
           try {
                root.setView(view, wparams, panelParentView);
           } catch (RuntimeException e) {

                  throw e;
           }  
   }
```
接下来，来查看[ViewRootImpl](http://androidxref.com/7.0.0_r1/xref/frameworks/base/core/java/android/view/ViewRootImpl.java)的setView():

```java 

 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
       synchronized (this) {
            if (mView == null) {
                  mView = view;
                  
                  // 开始加载布局
                  requestLayout();
                  
635                try {
636                    mOrigWindowType = mWindowAttributes.type;
637                    mAttachInfo.mRecomputeGlobalAttributes = true;
638                    collectViewAttributes();
                        //开始将窗口View对象添加到WMS中
639                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
640                            getHostVisibility(), mDisplay.getDisplayId(),
641                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
642                            mAttachInfo.mOutsets, mInputChannel);
643                } catch (RemoteException e) {
                           
                   }
                   
                   //... 接下来，检查添加结果
                  
            }
       
       }
 }
```


