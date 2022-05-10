### View的绘制流程

今天来讲讲View的绘制流程吧，这一块也是我比较薄弱的地方，而且之前没有什么耐心去看这一块。今天就来学习这一块的知识点。主要分为3部分：

+ View树的创建

+ `ViewRootImpl`的创建

+ 真正的绘制流程开始performTraversals



#### View树的创建

首先思考一个问题View的绘制从那里开始？关于这个问题我一开始也谷歌百度了很多东西，但是还是不能解决我的疑惑。所以今天我打算从根本上解决这个问题——从Activity的setContentView开始看。下面开始我的源码阅读过程。

##### Activity#setContentView

```java
public void setContentView(@LayoutRes int layoutResID) {
    //注释1 getWindow()返回的是一个PhoneWindow对象，所以下一步，就是要去PhoneWindow中去看看这里面做了什么。PhoneWindow也是Window类的唯一实现类
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

##### PhoneWindow#setContentView

```java
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    //注释2 mContentParent一开始的时候为空，所以会执行installDecor函数，mContextParent是一个ViewGroup对象
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
    mContentParentExplicitlySet = true;
}

//installDecor的注释很长，我们一步一步看
private void installDecor() {
    mForceDecorInstall = false;
    //注释3 mDecor是一个DecorView，这是窗口的顶层视图，包含窗口装饰。首次进来的时候mDecor是空的，所以获取执行generateDecor，在generateDecor里面会去创建一个DecorView，注意这个函数是在PhoneWindow里面的，也就是说PhoneWindow里面的mDecor有了初始值，那么就是DecorView与PhoneWindow建立了关联。
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    //注释4 注意这里mContentParent还是没有进行任何的初始化，所以还是为null，所以会执行generateLayout方法，而generateLayout的方法参数是上面干初始化的mDecor，而这个函数的返回值是给mContentParent赋值。
    if (mContentParent == null) {
        
        //这里面得到的是一个FrameLayout
        mContentParent = generateLayout(mDecor);
        
        //....省略未分析的代码
    }
}

```

##### PhoneWindow#generateLayout

```java
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.

  	//...省略很多代码，主要是获取Activity主题的一些属性，然后根据属性配置
    //注释5 这个默认情况下会为R.layout.screen_simple,screen_simple.xml的布局内容是一个LinearLayout里面包含两个子view，其中一个子view的id是content，并且是FrameLayout。
    int layoutResource;
    int features = getLocalFeatures();//获取Activity.java 代码的里面的配置，注意这里很上面的不一样，上面是根据Activity主题的配置，这里获取的是java代码里面的动态配置
    
    //...省略很多代码，
    mDecor.startChanging();
    //注释6 参数为一个LayoutInflate，和刚刚的布局文件id，这里面会调用mLayoutInflater的inflater创建view，然后调用DecorView的addView方法，也就是说刚才那个布局里面的控件都加入了DecorView中，也是就DecorView包含了他们
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

     //注释7 调用findViewById，其实调用的是DecorView的findViewById方法，然后返回一个ViewGroup。而这个id正好是注释5里面提到的content，所以这里面相当于返回了一个FrameLayout
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	
    //省略一些配置代码
    
	//注释8 这里面将FrameLayout返回
    return contentParent;
}
```

好分析到这里，很多人已经蒙了，什么跟什么啊，一点绘制的流程都没讲。确实这里面还是没有讲解到view真正去绘制的代码，但是这里面的知识点确实很重要。如果直接讲解view的绘制流程的代码，很多人会不知道为什么执行到了具体哪一个步骤。所以上面的流程步骤先耐心看完吧。我们现在有了什么？总结一下，我们有了`DecorView`，有了一个`FrameLayout`，还有一些可能配置的信息，比如TitleBar，他们的关系是怎么样的呢？如下图。

![image-20220421133753463](View%E7%9A%84%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B.assets/image-20220421133753463.png)

想一个问题，这里面是系统的层面的布局给改好了，我们传入的布局呢？这时候就需要回到`PhoneWindow`#setContentView里面了。

```java
public void setContentView(int layoutResID) {
	//省略已经分析过的代码

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                                                       getContext());
        transitionTo(newScene);
    } else {
        //最终会执行到这里,mContentParent也就是上面得到的id为content的frameLayout
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}

```

如图：

![image-20220421134238892](View%E7%9A%84%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B.assets/image-20220421134238892.png)

那么我们总结一下上面的流程和梳理一些对应的关系。（注意箭头的指向）

![image-20220421135327412](View%E7%9A%84%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B.assets/image-20220421135327412.png)

分析到这里整一个View树的创建过程就分析完了，看起来好像是`DecorView`是我们的最顶层的父类了，但是`DecorView`还有没有父类呢？答案是有的，它的顶层就是就是`ViewRootImpl`。

#### ViewRootImpl的创建

所以接下来我们就需要看一下`ViewRootImpl`是怎么和`DecorView`建立关系的

看一下`ViewRootImpl`类的描述：**视图层次结构的顶部，实现 `View` 和 `WindowManager` 之间所需的协议。这在很大程度上是 `WindowManagerGlobal` 的内部实现细节。**

`ViewRootImpl`的创建是在`WindowManagerGlobal`中的`addView`中创建的。而`WindowManagerGlobal`的`addView`是在`ActivityThread`中的`handleResumeActivity`中去调用的。`Activity`的生命周期都是由这个类管理的。这里我们直接看handleResumeActivity里面的方法

##### ActivityThread#handleResumeActivity

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                                 String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    //注释9 执行了这个方法之后，也就是Activity的onResume方法没执行了
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    if (r == null) {
        // We didn't actually resume the activity, so skipping any follow-up actions.
        return;
    }

    final Activity a = r.activity;

    if (localLOGV) {
        Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
               + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
    }

    final int forwardBit = isForward
        ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

    // If the window hasn't yet been added to the window manager,
    // and this guy didn't finish itself or start another activity,
    // then go ahead and add the window.
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        try {
            willBeVisible = ActivityManager.getService().willActivityBeVisible(
                a.getActivityToken());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    if (r.window == null && !a.mFinished && willBeVisible) {
        
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        //注释10 这里面的ViewManager实例时ViewManager
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
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                //注释11 将decor传入ViewManagerImpl的addView,addView，在ViewManagerImpl中的addView会去调用WindowManagerGlobal的addView方法，所以这里就会创建了ViewRootImpl
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }

    // Get rid of anything left hanging around.
    cleanUpPendingRemoveWindows(r, false /* force */);

    // The window is now visible if it has been added, we are not
    // simply finishing, and we are not starting another activity.
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        if (r.newConfig != null) {
            performConfigurationChangedForActivity(r, r.newConfig);
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                       + r.activity.mCurrentConfig);
            }
            r.newConfig = null;
        }
        if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
        WindowManager.LayoutParams l = r.window.getAttributes();
        if ((l.softInputMode
             & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
            != forwardBit) {
            l.softInputMode = (l.softInputMode
                               & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                | forwardBit;
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l);
            }
        }

        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }

    r.nextIdle = mNewActivities;
    mNewActivities = r;
    if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
    Looper.myQueue().addIdleHandler(new Idler());
}
```

##### WindowManagerGlobal#addView

```java
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
            && (context.getApplicationInfo().flags
                & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        // Start watching for system property changes.
        if (mSystemPropertyUpdater == null) {
            mSystemPropertyUpdater = new Runnable() {
                @Override public void run() {
                    synchronized (mLock) {
                        for (int i = mRoots.size() - 1; i >= 0; --i) {
                            mRoots.get(i).loadSystemProperties();
                        }
                    }
                }
            };
            SystemProperties.addChangeCallback(mSystemPropertyUpdater);
        }

        int index = findViewLocked(view, false);
        if (index >= 0) {
            if (mDyingViews.contains(view)) {
                // Don't wait for MSG_DIE to make it's way through root's queue.
                mRoots.get(index).doDie();
            } else {
                throw new IllegalStateException("View " + view
                                                + " has already been added to the window manager.");
            }
            // The previous removeView() had not completed executing. Now it has.
        }

        // If this is a panel window, then find the window it is being
        // attached to for future reference.
        if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
            wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            final int count = mViews.size();
            for (int i = 0; i < count; i++) {
                if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    panelParentView = mViews.get(i);
                }
            }
        }
        //注释12 这里会创建ViewRootImpl。
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            //注释13 并且这里会调用ViewRootImpl的setView方法
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

##### ViewRootImpl#setView

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

			//..,省略代码
            requestLayout();//里面会做两件重要的事，1检查是不是在子线程更新view，2、执行scheduleTraversals。而在scheduleTraversals中会执行一个异步任务，最终去执行performTraversals()，这个方法就是View绘制的开始了

			//..,省略代码,将ViewRootImpl设置为decorView的parent，这里执行完成之后，调用decorView.parent就有值了，就是ViewRootImpl
            view.assignParent(this);
      
        }
    }
}
```

#### 真正的绘制流程开始performTraversals

##### ViewRootImpl#performTraversals

这个方法很长，只要是调用了performMeasure，performLayout，performDraw

```java
    private void performTraversals() {
       	//省略代码..
        //如果第一次执行为
        if (mFirst) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            //省略代码..
        }
         
		//这里一般为true
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
			//省略代码..
            // Ask host how big it wants to be
            //这里就是对整一个布局树的测量了
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }


        if (!mStopped || mReportNextDraw) {
            // 执行performMeasure
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

        }
        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        boolean triggerGlobalLayoutListener = didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        //这里为true
        if (didLayout) {
            //这里就是对整一个布局树的布局
            performLayout(lp, mWidth, mHeight);

        }

        if (triggerGlobalLayoutListener) {
            mAttachInfo.mRecomputeGlobalAttributes = false;
            //这里的监听可以获取控件的大小了
            mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
        }

        if (!cancelDraw && !newSurface) {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }
			//执行绘制代码
            performDraw();
        } else {
            if (isViewVisible) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }

        mIsInTraversal = false;
    }

```

到这里View的绘制流程就已经到了我们最熟悉的三个函数里面了

`performTraversals`会依次调用`performMeasure`、`performLayout`和`performDraw`三个方法，这三个方法分别完成顶级View的measure、layout和draw这三大流程，其中在`performMeasure`中会调用measure方法，在measure方法中又会调用`onMeasure`方法，在`onMeasure`方法中则会对所有的子元素进行measure过程，这个时候`measure`流程就从父容器传递到子元素中了，这样就完成了一次`measure`过程。接着子元素会重复父容器的measure过程，如此反复就完成了整个View树的遍历。同理，`performLayout`和`performDraw`的 传递流程和`performMeasure`是类似的，唯一不同的是，`performDraw`的传递过程是在draw方法中通过`dispatchDraw`来实现的，不过这并没有本质区别。 

![image-20220421160901538](View%E7%9A%84%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B.assets/image-20220421160901538.png)

以上就是view的绘制流程了，至于viewGroup的各个方法实现的细节，这里就不展开来讲了。大体流程是这个个流程。后面讲解自定义ViewGroup的时候会讲解具体是怎么测量和怎么布局的。