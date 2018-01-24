# Activity的Window创建过程

Activity的启动过程会由ActivityThread中的performLaunchActivity方法来完成整个启动过程，在这个方法内部会通过类加载器创建Activity的实例对象，并调用其attach方法为其关联运行过程中所依赖的一系列上下文环境变量。

```
   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ...
        } catch (Exception e) {
            ...
        }
        if (activity != null) {
                ...
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
        }
        ...

        return activity;
    }


```

在Activity的attach方法里，系统会创建Activity所属的Window对象并为其设置回调接口，Window的创建是通过`new PhoneWindow(this, window, activityConfigCallback)`方法实现的。由于Activity实现了Window的Callback接口，因此当Window接收到外界的状态改变时就会回调Activity方法。CallBack接口中的方法很多，有一些是我们很熟悉的，onAttachedToWindow()、onDetachedFromWindow()、dispatchTouchEvent(MotionEvent event)。

```
   final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
    }

```


那么Activity的视图是如何附属在Window上的呢？

通过activity的setContentView方法我们可以看到：

```
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

```

Activity将具体实现交给了Window处理，而Window的具体实现是[PhoneWindow](https://android.googlesource.com/platform/frameworks/base/+/android-6.0.1_r25/core/java/com/android/internal/policy/PhoneWindow.java)。

```
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }

```

PhoneWindow的setContentWindow方法大致遵循如下几个步骤：

1.如果没有DecorView，那么就创建它

```
 private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        }
         if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            ...
        }
  }
```

DecorView是一个FrameLayout，它是Activity中的顶级View，一般来说它的内部包括标题栏和内容栏，但是这个会随着主题的变换而改变。不管怎样，内容栏是一定要存在的，并且内容具有固定的id，`android.R.id.content`。
为了初始化DecorView的结构，PhoneWindow还需要通过generateLayout方法来加载具体的布局文件到DecorView中，具体的代码如下：

```
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
        return contentParent;

```


2.将View添加到DecorView的mContentParent中

`mLayoutInflater.inflate(layoutResID, mContentParent);`


3.回调Activity的onContentChanged方法通知Activity视图已经发生改变

经过了上面的三个步骤，DecorView已经被创建并初始化完毕，Activity的布局文件也已经成功添加到DecorView的mContentParent中，但是这个时候DecorView还没有被WindowManager正式添加到Window中。在ActivityThread的handleResumeActivity中，首先会调用Activity的onResume方法，接着会调用Activity的makeVisible()，正是在makeVisible方法中，DecorView真正地完成了添加和显示这两个过程。

```
    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }

```
