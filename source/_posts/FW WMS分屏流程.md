---

title: Android 13 WMS分屏流程
date: 2023-9-21 02:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

#### 分屏启动

##### 桌面入口部分

桌面点击这个Split top按钮的onClick后触发自己进程的view相关的动画，对应堆栈：

```
at com.android.quickstep.views.RecentsView.createInitialSplitSelectAnimation(RecentsView.java:2769)
at com.android.quickstep.views.RecentsView.createTaskDismissAnimation(RecentsView.java:2976)
at com.android.quickstep.views.RecentsView.createSplitSelectInitAnimation(RecentsView.java:4023)
at com.android.launcher3.uioverrides.RecentsViewStateController.handleSplitSelectionState(RecentsViewStateController.java:120)
at com.android.launcher3.uioverrides.RecentsViewStateController.setStateWithAnimationInternal(RecentsViewStateController.java:96)
at com.android.launcher3.uioverrides.BaseRecentsViewStateController.setStateWithAnimation(BaseRecentsViewStateController.java:87)
at com.android.launcher3.uioverrides.BaseRecentsViewStateController.setStateWithAnimation(BaseRecentsViewStateController.java:53)
at com.android.launcher3.statemanager.StateManager.createAnimationToNewWorkspaceInternal(StateManager.java:324)
at com.android.launcher3.statemanager.StateManager.goToStateAnimated(StateManager.java:259)
at com.android.launcher3.statemanager.StateManager.goToState(StateManager.java:247)
at com.android.launcher3.statemanager.StateManager.goToState(StateManager.java:150)
at com.android.launcher3.statemanager.StateManager.goToState(StateManager.java:143)
at com.android.quickstep.views.LauncherRecentsView.initiateSplitSelect(LauncherRecentsView.java:182)
at com.android.quickstep.views.TaskView.initiateSplitSelect(TaskView.java:1541)
at com.android.quickstep.TaskShortcutFactory$SplitSelectSystemShortcut.onClick(TaskShortcutFactory.java:132)
at com.android.quickstep.views.TaskMenuView.lambda$addMenuOption$1(TaskMenuView.java:254)
at com.android.quickstep.views.TaskMenuView$$ExternalSyntheticLambda2.run(Unknown Source:4)
at com.android.quickstep.views.RecentsView.finishRecentsAnimation(RecentsView.java:4536)
at com.android.quickstep.views.TaskMenuView.lambda$addMenuOption$2(TaskMenuView.java:252)
at com.android.quickstep.views.TaskMenuView$$ExternalSyntheticLambda1.run(Unknown Source:6)
at com.android.quickstep.views.RecentsView.switchToScreenshot(RecentsView.java:4962)
at com.android.quickstep.views.TaskMenuView.lambda$addMenuOption$3$com-android-quickstep-views-TaskMenuView(TaskMenuView.java:251)
at com.android.quickstep.views.TaskMenuView$$ExternalSyntheticLambda5.onClick(Unknown Source:4)
```

点击之后会有相关动画的准备工作：
比如选择短信，短信app会放到上面，下面多任务依旧是多任务，但是位置发生了变化。

createInitialSplitSelectAnimation是上面那个短信应用上半屏幕部分显示的动画构建

```java
/**
     * Places an {@link FloatingTaskView} on top of the thumbnail for {@link #mSplitHiddenTaskView}
     * and then animates it into the split position that was desired
     */
    private void createInitialSplitSelectAnimation(PendingAnimation anim) {
        mOrientationHandler.getInitialSplitPlaceholderBounds(mSplitPlaceholderSize,
                mSplitPlaceholderInset, mActivity.getDeviceProfile(),
                mSplitSelectStateController.getActiveSplitStagePosition(), mTempRect);

        RectF startingTaskRect = new RectF();
        if (mSplitHiddenTaskView != null) {
            mSplitHiddenTaskView.setVisibility(INVISIBLE);//把TaskView设置不可见
            //要创建一个TaskView的副本，并且要把这个副本进行相关的动画播放显示在最顶端一个小区域
            mFirstFloatingTaskView = FloatingTaskView.getFloatingTaskView(mActivity,
                    mSplitHiddenTaskView.getThumbnail(),
                    mSplitHiddenTaskView.getThumbnail().getThumbnail(),
                    mSplitHiddenTaskView.getIconView().getDrawable(), startingTaskRect);
            mFirstFloatingTaskView.setAlpha(1);
            //根据相关的初始化尺寸来构建对应anim
            mFirstFloatingTaskView.addAnimation(anim, startingTaskRect, mTempRect,
                    true /* fadeWithThumbnail */, true /* isStagedTask */);
        } else {
         
        }
        anim.addEndListener(success -> {
            if (success) {
                mSplitToast.show();
                //动画播放完成后，显示出一个Toast "点按另一个应用即可使用分屏"
            }
        });
    }

```

此时分屏在桌面多任务完成了第一步，把TaskView放到顶部，接下来要手动选择一个下屏app，才可以构成真正意义的分屏。

底部点击TaskView触发动画流程

```java
 private void onClick(View view) {
        if (confirmSecondSplitSelectApp()) {
            return;
        }
      	launchTasks();
}
 private boolean confirmSecondSplitSelectApp() {
        int index = getChildTaskIndexAtPosition(mLastTouchDownPosition);
        TaskIdAttributeContainer container = mTaskIdAttributeContainer[index];
        //调用到了RecentsView的confirmSplitSelect方法
        return getRecentsView().confirmSplitSelect(this, container.getTask(),
                container.getIconView(), container.getThumbnailView());
    }
//RecentsView.java
	//方法进行相关的动画准备和启动
     public boolean confirmSplitSelect(TaskView containerTaskView, Task task, IconView iconView,
            TaskThumbnailView thumbnailView) {
        ...
        mSplitSelectStateController.setSecondTask(task);//设置底部区域要显示的task
        RectF secondTaskStartingBounds = new RectF();//初始化底部的区域
        Rect secondTaskEndingBounds = new Rect();
        Rect firstTaskStartingBounds = new Rect();//初始化顶部的区域
        Rect firstTaskEndingBounds = mTempRect;
        //获取动画时间
        int duration = mActivity.getStateManager().getState().getTransitionDuration(mActivity,
                false /* isToState */);
        PendingAnimation pendingAnimation = new PendingAnimation(duration);

        int halfDividerSize = getResources()
                .getDimensionPixelSize(R.dimen.multi_window_task_divider_size) / 2;
        //获取对应的一个上屏幕区域和下屏幕区域截至区域firstTaskEndingBounds，secondTaskEndingBounds
        mOrientationHandler.getFinalSplitPlaceholderBounds(halfDividerSize,
                mActivity.getDeviceProfile(),
                mSplitSelectStateController.getActiveSplitStagePosition(), firstTaskEndingBounds,
                secondTaskEndingBounds);
		
		//获取当前的上屏幕区域坐标情况放入firstTaskStartingBounds
        mFirstFloatingTaskView.getBoundsOnScreen(firstTaskStartingBounds);
        //有了初始区域和结束区域，进行相关的动画配置
        mFirstFloatingTaskView.addAnimation(pendingAnimation,
                new RectF(firstTaskStartingBounds), firstTaskEndingBounds,
                false /* fadeWithThumbnail */, true /* isStagedTask */);

		//下屏部分
        mSecondFloatingTaskView = FloatingTaskView.getFloatingTaskView(mActivity,
                thumbnailView, thumbnailView.getThumbnail(),
                iconView.getDrawable(), secondTaskStartingBounds);
        mSecondFloatingTaskView.setAlpha(1);
        mSecondFloatingTaskView.addAnimation(pendingAnimation, secondTaskStartingBounds,
                secondTaskEndingBounds, true /* fadeWithThumbnail */, false /* isStagedTask */);
          //进行动画结束监听，在动画结束时候才进行launchSplitTasks，正式调用到systemui启动真正分屏
        pendingAnimation.addEndListener(aBoolean -> {
            mSplitSelectStateController.launchSplitTasks(
                    aBoolean1 -> RecentsView.this.resetFromSplitSelectionState());
        });
        if (containerTaskView.containsMultipleTasks()) {
            // If we are launching from a child task, then only hide the thumbnail itself
            mSecondSplitHiddenView = thumbnailView;
        } else {
            mSecondSplitHiddenView = containerTaskView;
        }
        mSecondSplitHiddenView.setVisibility(INVISIBLE);//下屏的TaskView要被隐藏
        pendingAnimation.buildAnim().start();//启动正式动画
        return true;
    }

```

动画结束后执行mSplitSelectStateController.launchSplitTasks，正式调用systemui相关的shell接口进行操作真正的task相关分屏业务

```
at com.android.quickstep.SystemUiProxy.startTasksWithLegacyTransition(SystemUiProxy.java:621)
                                                                                                    	at com.android.quickstep.util.SplitSelectStateController.launchTasks(SplitSelectStateController.java:201)
                                                                                                    	at com.android.quickstep.util.SplitSelectStateController.launchSplitTasks(SplitSelectStateController.java:124)
                                                                                                    	at com.android.quickstep.views.RecentsView.lambda$confirmSplitSelect$24$com-android-quickstep-views-RecentsView(RecentsView.java:4082)
```

##### SystemUI部分

```
 //SystemUiProxy.java
 private ISplitScreen mSplitScreen;
  public void startTasksWithLegacyTransition(int mainTaskId, Bundle mainOptions, int sideTaskId,
            Bundle sideOptions, @SplitConfigurationOptions.StagePosition int sidePosition,
            float splitRatio, RemoteAnimationAdapter adapter) {
        mSplitScreen.startTasksWithLegacyTransition(mainTaskId, mainOptions, sideTaskId,
                        sideOptions, sidePosition, splitRatio, adapter);
        }
    }
```

mSplitScreen是ISplitScreen  Binder接口,会调用到SystemUI所在的服务端
这里对应SystemUI端的代码：

```java
 //frameworks/base/libs/WindowManager/Shell/src/com/android/wm/shell/splitscreen/SplitScreenController.java
 private static class ISplitScreenImpl extends ISplitScreen.Stub {

        @Override
        public void startTasksWithLegacyTransition(int mainTaskId, @Nullable Bundle mainOptions,
                int sideTaskId, @Nullable Bundle sideOptions, @SplitPosition int sidePosition,
                float splitRatio, RemoteAnimationAdapter adapter) {
                //注意这里有线程切换哈，binder线程到main线程
            executeRemoteCallWithTaskPermission(mController, "startTasks",
                    (controller) -> controller.mStageCoordinator.startTasksWithLegacyTransition(
                            mainTaskId, mainOptions, sideTaskId, sideOptions, sidePosition,
                            splitRatio, adapter));
        }
}
//frameworks/base/libs/WindowManager/Shell/src/com/android/wm/shell/splitscreen/StageCoordinator.java
  private void startWithLegacyTransition(int mainTaskId, int sideTaskId,
            @Nullable PendingIntent pendingIntent, @Nullable Intent fillInIntent,
            @Nullable Bundle mainOptions, @Nullable Bundle sideOptions,
            @SplitPosition int sidePosition, float splitRatio, RemoteAnimationAdapter adapter) {
     	//1、初始化分屏的分割线的布局
        mSplitLayout.init();
    	//初始化用来跨进程传递的WindowContainerTransaction
        final WindowContainerTransaction wct = new WindowContainerTransaction();
    
   		//初始化一个远程动画的运行回调binder对象
        IRemoteAnimationRunner wrapper = new IRemoteAnimationRunner.Stub() {
            @Override
            public void onAnimationStart(@WindowManager.TransitionOldType int transit,
                    RemoteAnimationTarget[] apps,
                    RemoteAnimationTarget[] wallpapers,
                    RemoteAnimationTarget[] nonApps,
                    final IRemoteAnimationFinishedCallback finishedCallback) {
            ...
        };
        //包装一下成为RemoteAnimationAdapter类型
        RemoteAnimationAdapter wrappedAdapter = new RemoteAnimationAdapter(
                wrapper, adapter.getDuration(), adapter.getStatusBarTransitionDelay());
		...
		//构建出对应的mainOptions，及上分屏的启动option
         mainOptions = mainActivityOptions.toBundle();
 		//准备好对应的sideOptions，下分屏的option
        sideOptions = sideOptions != null ? sideOptions : new Bundle();
        setSideStagePosition(sidePosition, wct);
		//设置分界线比例，一般大小上下屏幕都一样大，为0.5，这里就把对应的上下分屏的bound计算出来了
        mSplitLayout.setDivideRatio(splitRatio);
        if (!mMainStage.isActive()) {//设置为active
            mMainStage.activate(wct, false /* reparent */);
        }
        //把计算出来的上下分屏的bound都设置给对应的configration，传递到systemserver端，然后systemserver更新task的bound
        updateWindowBounds(mSplitLayout, wct);
        //2、需要让装载分屏的roottask进行reorder，主要就是为了把分屏移到最前面, 注意这里的mRootTaskInfo其实就是一开始SystemUI负责创建的mutilwindow的task
        wct.reorder(mRootTaskInfo.token, true);

		//准备好对应的option参数
        // Make sure the launch options will put tasks in the corresponding split roots
        addActivityOptions(mainOptions, mMainStage);
        addActivityOptions(sideOptions, mSideStage);

		//3、分别准备好对应上下分屏启动task的transition
        // Add task launch requests
        wct.startTask(mainTaskId, mainOptions);
        if (withIntent) {
            wct.sendPendingIntent(pendingIntent, fillInIntent, sideOptions);
        } else {
            wct.startTask(sideTaskId, sideOptions);
        }
        //最后把前面准备好的WindowContainerTranstion统一进行apply到systemserver
        // Using legacy transitions, so we can't use blast sync since it conflicts.
        mTaskOrganizer.applyTransaction(wct);
        mSyncQueue.runInSync(t -> {
        	//这里主要进行相关的divider分界线显示
            setDividerVisibility(true, t);
            updateSurfaceBounds(mSplitLayout, t, false /* applyResizingOffset */);
        });
    }


```

mMainStage和mSideStage属于和RootTask一样，分屏的RootTask一般会带两个子task，分别是mMainStage和mSideStage的RootTaskInfo。

上面注释中的代码主要做了下面几件事情：

1. 分割线初始化和上下分屏的区域参数
2. 分屏的RootTask进行reorder，移到最前面
3. 上下分屏执行startTask

```java
 //SplitLayout的init
public void init() {
        if (mInitialized) return;
        mInitialized = true;
    //调用mSplitWindowManager初始化，主要是创建对应window出来,不是用普通的windowmanager创建的，dumpsys window windows是看不到的，通过dumpsys SurfaceFlinger可以看到
        mSplitWindowManager.init(this, mInsetsState);
        mDisplayImeController.addPositionProcessor(mImePositionProcessor);
    }
//SplitWindowManager的init
  void init(SplitLayout splitLayout, InsetsState insetsState) {
        mViewHost = new SurfaceControlViewHost(mContext, mContext.getDisplay(), this);
        mDividerView = (DividerView) LayoutInflater.from(mContext)
                .inflate(R.layout.split_divider, null /* root */);

        final Rect dividerBounds = splitLayout.getDividerBounds();
        WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
                dividerBounds.width(), dividerBounds.height(), TYPE_DOCK_DIVIDER,
                FLAG_NOT_FOCUSABLE | FLAG_NOT_TOUCH_MODAL | FLAG_WATCH_OUTSIDE_TOUCH
                        | FLAG_SPLIT_TOUCH | FLAG_SLIPPERY,
                PixelFormat.TRANSLUCENT);
        lp.token = new Binder();
        lp.setTitle(mWindowName);
        lp.privateFlags |= PRIVATE_FLAG_NO_MOVE_ANIMATION | PRIVATE_FLAG_TRUSTED_OVERLAY;
        mViewHost.setView(mDividerView, lp);
        mDividerView.setup(splitLayout, this, mViewHost, insetsState);
    }
```

区域的计算确定部分，setDivideRatio方法

```java
public void setDivideRatio(float ratio) {
	//这里的position是非常关键，这个是roottask上下屏幕相等情况
    final int position = isLandscape()
            ? mRootBounds.left + (int) (mRootBounds.width() * ratio)
            : mRootBounds.top + (int) (mRootBounds.height() * ratio);
    final DividerSnapAlgorithm.SnapTarget snapTarget =
            mDividerSnapAlgorithm.calculateNonDismissingSnapTarget(position);
    //设置对应的位置position，非常关键会触发bound的计算
    setDividePosition(snapTarget.position, false /* applyLayoutChange */);
}
   void setDividePosition(int position, boolean applyLayoutChange) {
        mDividePosition = position;
       //根据新新的mDividePosition计算新的bound
        updateBounds(mDividePosition);
        if (applyLayoutChange) {
            mSplitLayoutHandler.onLayoutSizeChanged(this);
        }
    }
       /** Updates recording bounds of divider window and both of the splits. */
    private void updateBounds(int position) {
        mDividerBounds.set(mRootBounds);
        mBounds1.set(mRootBounds);
        mBounds2.set(mRootBounds);
        final boolean isLandscape = isLandscape(mRootBounds);
        if (isLandscape) {
            position += mRootBounds.left;
            mDividerBounds.left = position - mDividerInsets;
            mDividerBounds.right = mDividerBounds.left + mDividerWindowWidth;
            mBounds1.right = position;
            mBounds2.left = mBounds1.right + mDividerSize;
        } else {
            position += mRootBounds.top;
            mDividerBounds.top = position - mDividerInsets;
            mDividerBounds.bottom = mDividerBounds.top + mDividerWindowWidth;
            mBounds1.bottom = position;
            mBounds2.top = mBounds1.bottom + mDividerSize;
        }
        DockedDividerUtils.sanitizeStackBounds(mBounds1, true /** topLeft */);
        DockedDividerUtils.sanitizeStackBounds(mBounds2, false /** topLeft */);
        mSurfaceEffectPolicy.applyDividerPosition(position, isLandscape);
    }

```

已经计算好了分屏的bound后，需要把bound设置到WindowContainerTransition中进行传递：

updateWindowBounds(mSplitLayout, wct);

```java
    private void updateWindowBounds(SplitLayout layout, WindowContainerTransaction wct) {
        final StageTaskListener topLeftStage =
                mSideStagePosition == SPLIT_POSITION_TOP_OR_LEFT ? mSideStage : mMainStage;
        final StageTaskListener bottomRightStage =
                mSideStagePosition == SPLIT_POSITION_TOP_OR_LEFT ? mMainStage : mSideStage;
        //分别有了上下屏task信息后，要对这些task的bound进行修改，applyTaskChanges就是关键的方法（这里的task还不是直接app的task，还是分屏的mMainStage及mSideStage对应的容器task）
        layout.applyTaskChanges(wct, topLeftStage.mRootTaskInfo, bottomRightStage.mRootTaskInfo);
    }
      /** Apply recorded task layout to the {@link WindowContainerTransaction}. */
    public void applyTaskChanges(WindowContainerTransaction wct,
            ActivityManager.RunningTaskInfo task1, ActivityManager.RunningTaskInfo task2) {
        if (!mBounds1.equals(mWinBounds1) || !task1.token.equals(mWinToken1)) {
            //WindowContainerTransaction设置好对应的task的bound数据
            wct.setBounds(task1.token, mBounds1);
            wct.setSmallestScreenWidthDp(task1.token, getSmallestWidthDp(mBounds1));
            mWinBounds1.set(mBounds1);
            mWinToken1 = task1.token;
        }
        if (!mBounds2.equals(mWinBounds2) || !task2.token.equals(mWinToken2)) {
            wct.setBounds(task2.token, mBounds2);
            wct.setSmallestScreenWidthDp(task2.token, getSmallestWidthDp(mBounds2));
            mWinBounds2.set(mBounds2);
            mWinToken2 = task2.token;
        }
    }
    /**
     * Resize a container.
     */
    @NonNull
    public WindowContainerTransaction setBounds(
            @NonNull WindowContainerToken container,@NonNull Rect bounds) {
        //Change的构造，bounds变化靠Change变量进行传递
        Change chg = getOrCreateChange(container.asBinder());
        chg.mConfiguration.windowConfiguration.setBounds(bounds);
        chg.mConfigSetMask |= ActivityInfo.CONFIG_WINDOW_CONFIGURATION;
        chg.mWindowSetMask |= WindowConfiguration.WINDOW_CONFIG_BOUNDS;
        return this;
    }
```

 对分屏的RootTask进行reorder

```java
  @NonNull
    public WindowContainerTransaction reorder(@NonNull WindowContainerToken child, boolean onTop) {
        mHierarchyOps.add(HierarchyOp.createForReorder(child.asBinder(), onTop));
        return this;
    }
    	//创建对应的HIERARCHY_OP_TYPE_REORDER的HierarchyOp进行传递
         public static HierarchyOp createForReorder(@NonNull IBinder container, boolean toTop) {
            return new HierarchyOp.Builder(HIERARCHY_OP_TYPE_REORDER)
                    .setContainer(container)
                    .setReparentContainer(container)
                    .setToTop(toTop)
                    .build();
        }

```

对分屏Task进行startTask

```java

public WindowContainerTransaction startTask(int taskId, @Nullable Bundle options) {
        mHierarchyOps.add(HierarchyOp.createForTaskLaunch(taskId, options));
        return this;
    }
     /** Create a hierarchy op for launching a task. */
        public static HierarchyOp createForTaskLaunch(int taskId, @Nullable Bundle options) {
            final Bundle fullOptions = options == null ? new Bundle() : options;
            fullOptions.putInt(LAUNCH_KEY_TASK_ID, taskId);
            return new HierarchyOp.Builder(HIERARCHY_OP_TYPE_LAUNCH_TASK)
                    .setToTop(true)
                    .setLaunchOptions(fullOptions)
                    .build();
        }
```

##### SystemServer部分

mTaskOrganizer.applyTransaction(wct);跨进程会调用到WindowOrganizerController的applyTransaction

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java
 private void applyTransaction(@NonNull WindowContainerTransaction t, int syncId,
            @Nullable Transition transition, @NonNull CallerInfo caller,
            @Nullable Transition finishTransition) {
        try {
          //获取transition的change部分
            Iterator<Map.Entry<IBinder, WindowContainerTransaction.Change>> entries =
                    t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
				//获取一个的change，change中包裹的具体WindowContainer，上屏和下屏的对应Task
                int containerEffect = applyWindowContainerChange(wc, entry.getValue(),
                        t.getErrorCallbackToken());
           
            }
            // Hierarchy changes
            final List<WindowContainerTransaction.HierarchyOp> hops = t.getHierarchyOps();
            final int hopSize = hops.size();
            if (hopSize > 0) {
                final boolean isInLockTaskMode = mService.isInLockTaskMode();
                for (int i = 0; i < hopSize; ++i) {
                //对reorder和starTask的操作进行处理
                    effects |= applyHierarchyOp(hops.get(i), effects, syncId, transition,
                            isInLockTaskMode, caller, t.getErrorCallbackToken(),
                            t.getTaskFragmentOrganizer(), finishTransition);
                }
            }
           ...
    }

```

总结一下systemserver端处理其实和systemui客户端一样的：
1、针对区域大小bounds的变化，对这个Change相关进行applyWindowContainerChange方法来进行处理
2、针对task相关的2类操作，reorder和startTask两类都是包装成了HierarchyOp，调用applyHierarchyOp方法来进行处理

**区域大小bounds变化applyWindowContainerChange**

```java

 private int applyWindowContainerChange(WindowContainer wc,
            WindowContainerTransaction.Change c, @Nullable IBinder errorCallbackToken) {
        int effects = applyChanges(wc, c, errorCallbackToken);
        if (wc instanceof DisplayArea) {
            effects |= applyDisplayAreaChanges(wc.asDisplayArea(), c);
        } else if (wc instanceof Task) {
            effects |= applyTaskChanges(wc.asTask(), c);
        }
        return effects;
    }
    private int applyChanges(WindowContainer<?> container,
            WindowContainerTransaction.Change change, @Nullable IBinder errorCallbackToken) {
        //获取change的configration，bounds变化被包在了configration里面，再调用container的进行通知configration的变化
                final Configuration c =
                        new Configuration(container.getRequestedOverrideConfiguration());
                c.setTo(change.getConfiguration(), configMask, windowMask);
                //把对应的bounds设置给了Task
                container.onRequestedOverrideConfigurationChanged(c);         
        return effects;
    }
```

**applyHierarchyOp**

```java
 private int applyHierarchyOp(WindowContainerTransaction.HierarchyOp hop, int effects,
            int syncId, @Nullable Transition transition, boolean isInLockTaskMode,
            @NonNull CallerInfo caller, @Nullable IBinder errorCallbackToken,
            @Nullable ITaskFragmentOrganizer organizer, @Nullable Transition finishTransition) {
        final int type = hop.getType();
          switch (type) {
 				 case HIERARCHY_OP_TYPE_LAUNCH_TASK: {
            
            //这个launchOpts就是systemui传递的mainOptions和sideOptions，launchOpts里面的关键数据KEY_LAUNCH_ROOT_TASK_TOKEN，即stage.mRootTaskInfo.token
                final Bundle launchOpts = hop.getLaunchOptions();
                //注意这里是获取具体要启动app的taskId，不是上下分屏的id
                final int taskId = launchOpts.getInt(
                        WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                //转化成SafeActivityOptions类型
                final SafeActivityOptions safeOptions =
                        SafeActivityOptions.fromBundle(launchOpts, caller.mPid, caller.mUid)                 
                //这里有个线程切换操作，而且还会等执行完成        
                waitAsyncStart(() -> mService.mTaskSupervisor.startActivityFromRecents(
                        caller.mPid, caller.mUid, taskId, safeOptions));
                break;
            }
        }
     }
```

```java

int startActivityFromRecents(int callingPid, int callingUid, int taskId,
            SafeActivityOptions options) {
        final Task task;
        final int taskCallingUid;
        final String callingPackage;
        final String callingFeatureId;
        final Intent intent;
        final int userId;
        //获取传递进来的options，获取上下分屏的容器task
        final ActivityOptions activityOptions = options != null
                ? options.getOptions(this)
                : null;
        boolean moveHomeTaskForward = true;
        synchronized (mService.mGlobalLock) {
            int activityType = ACTIVITY_TYPE_UNDEFINED;
            if (activityOptions != null) {
                activityType = activityOptions.getLaunchActivityType();
              ... 
           //通过taskId来获取一个Task，把taskId对应的Task进行reparent到新上下分屏的容器Task，实现层级结构树上面的挂载完成，剩下就是一系列操作来保证Activiyt生命周期正常         
                task = mRootWindowContainer.anyTaskForId(taskId,
                        MATCH_ATTACHED_TASK_OR_RECENT_TASKS_AND_RESTORE, activityOptions, ON_TOP);
              //获取了task之后进行相关的Activity操作
                if (!mService.mAmInternal.shouldConfirmCredentials(task.mUserId)
                        && task.getRootActivity() != null) {
                        //获取task的Activity
                    final ActivityRecord targetActivity = task.getTopNonFinishingActivity();             
                    try {
                    //这里非常关键的把task移动到前台，在里面会对focus和resume进行设置
                        mService.moveTaskToFrontLocked(null /* appThread */,
                                null /* callingPackage */, task.mTaskId, 0, options);
                      ...
                    }
                    
                    
 Task anyTaskForId(int id, @RootWindowContainer.AnyTaskForIdMatchTaskMode int matchMode,
            @Nullable ActivityOptions aOptions, boolean onTop) {
         final PooledPredicate p = PooledLambda.obtainPredicate(
                Task::isTaskId, PooledLambda.__(Task.class), id);
        Task task = getTask(p);//遍历获取Task
        p.recycle();

        if (task != null) {
            if (aOptions != null) {
             	//这里有调用了getOrCreateRootTask来获取targetRootTask
                final Task targetRootTask =
                        getOrCreateRootTask(null, aOptions, task, onTop);
                if (targetRootTask != null && task.getRootTask() != targetRootTask) {
                    final int reparentMode = onTop
                            ? REPARENT_MOVE_ROOT_TASK_TO_FRONT : REPARENT_LEAVE_ROOT_TASK_IN_PLACE;
                    task.reparent(targetRootTask, onTop, reparentMode, ANIMATE, DEFER_RESUME,
                            "anyTaskForId");
                }
            }
            return task;
        }
		...
    }
    
    Task getOrCreateRootTask(@Nullable ActivityRecord r,
            @Nullable ActivityOptions options, @Nullable Task candidateTask,
            @Nullable Task sourceTask, boolean onTop,
            @Nullable LaunchParamsController.LaunchParams launchParams, int launchFlags) {
        // First preference goes to the launch root task set in the activity options.
        if (options != null) {
        //systemui传递的mainoptions
            final Task candidateRoot = Task.fromWindowContainerToken(options.getLaunchRootTask());
            if (candidateRoot != null && canLaunchOnDisplay(r, candidateRoot)) {
                return candidateRoot;//直接返回了options带的task
            }
        }
        ...
        }

```

再看下最重要的moveTaskToFrontLocked方法：

```java

 void moveTaskToFrontLocked(@Nullable IApplicationThread appThread,
            @Nullable String callingPackage, int taskId, int flags, SafeActivityOptions options) {
        ...
        try {
            final Task task = mRootWindowContainer.anyTaskForId(taskId);
            ...
            ActivityOptions realOptions = options != null
                    ? options.getOptions(mTaskSupervisor)
                    : null;
             //寻找到task而且移到最前端
            mTaskSupervisor.findTaskToMoveToFront(task, flags, realOptions, "moveTaskToFront",
                    false /* forceNonResizable */);
				
			//开始启动StartingWindow
            final ActivityRecord topActivity = task.getTopNonFinishingActivity();
            if (topActivity != null) {
                // We are reshowing a task, use a starting window to hide the initial draw delay
                // so the transition can start earlier.
                topActivity.showStartingWindow(true /* taskSwitch */);
            }
		...
    }
     
     
/** This doesn't just find a task, it also moves the task to front. */
    void findTaskToMoveToFront(Task task, int flags, ActivityOptions options, String reason,
            boolean forceNonResizeable) {
        //这里currentRootTask是根task
        Task currentRootTask = task.getRootTask();
        ...
            final ActivityRecord r = task.getTopNonFinishingActivity();
            //这里又调用到了关键moveTaskToFront方法
            currentRootTask.moveTaskToFront(task, false /* noAnimation */, options,
                    r == null ? null : r.appTimeTracker, reason);
    }

	final void moveTaskToFront(Task tr, boolean noAnimation, ActivityOptions options,
            AppTimeTracker timeTracker, boolean deferResume, String reason) {
      		...

            // Set focus to the top running activity of this task and move all its parents to top.
            //调用到了顶部ActivityRecord的moveFocusableActivityToTop方法
            top.moveFocusableActivityToTop(reason);
			...
            if (!deferResume) {//进行对应resume操作
                mRootWindowContainer.resumeFocusedTasksTopActivities();
            }
       ...
    }
     
	boolean moveFocusableActivityToTop(String reason) {
        final Task rootTask = getRootTask();
   		//rootTask把当前app的task移到最前
        rootTask.moveToFront(reason, task);
        // Report top activity change to tracking services and WM
        if (mRootWindowContainer.getTopResumedActivity() == this) { 
          //下面还需要设置，是因为getTopResumedActivity可能为null，还会通过getFocusedActivity作为ResumedActivity
          //把ActivityRecord变成Resumed状态
            mAtmService.setResumedActivityUncheckLocked(this, reason);
        }
        return true;
    }
     
   @Nullable
    ActivityRecord getTopResumedActivity() {
        final Task focusedRootTask = getTopDisplayFocusedRootTask();
        final ActivityRecord resumedActivity = focusedRootTask.getTopResumedActivity();
        if (resumedActivity != null && resumedActivity.app != null) {
            return resumedActivity;
        }
        // The top focused root task might not have a resumed activity yet - look on all displays in
        // focus order.
        //resumedActivity为null后获取getFocusedActivity
        return getItemFromTaskDisplayAreas(TaskDisplayArea::getFocusedActivity);
    }
```

#### 分割线

##### 分割线创建

DividerSnapAlgorithm是专门管理分割线位置相关的算法类

frameworks/base/core/java/com/android/internal/policy/DividerSnapAlgorithm.java

分割线的代表类SnapTarget

```java
//DividerSnapAlgorithm内部类
    /**
     * Represents a snap target for the divider.
     */
    public static class SnapTarget {
        public static final int FLAG_NONE = 0;

        /** If the divider reaches this value, the left/top task should be dismissed. */
        public static final int FLAG_DISMISS_START = 1;

        /** If the divider reaches this value, the right/bottom task should be dismissed */
        public static final int FLAG_DISMISS_END = 2;
		//表分割线位置
        /** Position of this snap target. The right/bottom edge of the top/left task snaps here. */
        public final int position;

        /**
         * Like {@link #position}, but used to calculate the task bounds which might be different
         * from the stack bounds.
         */
        public final int taskPosition;
		//区分边际分割线，主要有FLAG_DISMISS_START（上）和FLAG_DISMISS_END（下），其他都是FLAG_NONE
        public final int flag;
		
        public boolean isMiddleTarget;

        /**
         * Multiplier used to calculate distance to snap position. The lower this value, the harder
         * it's to snap on this target
         */
        //这个属于一个放大因子，针对首尾分割线距离的放大，让首尾分割线不会因为误触进入
        private final float distanceMultiplier;
    }
```

计算分割线方法如下

```java
private void calculateTargets(boolean isHorizontalDivision, int dockedSide) {
        mTargets.clear();
        int dividerMax = isHorizontalDivision
                ? mDisplayHeight
                : mDisplayWidth;
        int startPos = -mDividerSize;
        if (dockedSide == DOCKED_RIGHT) {
            startPos += mInsets.left;
        }
        //首先添加SnapTarget.FLAG_DISMISS_START的SnapTarget
        mTargets.add(new SnapTarget(startPos, startPos, SnapTarget.FLAG_DISMISS_START,
                0.35f));
        switch (mSnapMode) {//这里需要根据mSnapMode来分别创建分割情况
            case SNAP_MODE_16_9://正常竖屏就是SNAP_MODE_16_9，一般会创建3个分割线
                addRatio16_9Targets(isHorizontalDivision, dividerMax);
                break;
            case SNAP_FIXED_RATIO:
                addFixedDivisionTargets(isHorizontalDivision, dividerMax);
                break;
            case SNAP_ONLY_1_1://正常横屏就是SNAP_ONLY_1_1，所以只有中间一个分割线
                addMiddleTarget(isHorizontalDivision);
                break;
            case SNAP_MODE_MINIMIZED:
                addMinimizedTarget(isHorizontalDivision, dockedSide);
                break;
        }
        
        //最后添加SnapTarget.FLAG_DISMISS_END的SnapTarget
        mTargets.add(new SnapTarget(dividerMax, dividerMax, SnapTarget.FLAG_DISMISS_END, 0.35f));
    }

```

参数isHorizontalDivision，代表是当前分割线是横着显示还是竖着显示，竖屏分割线是横着显示isHorizontalDivision =true

```java
  public DividerSnapAlgorithm(Resources res, int displayWidth, int displayHeight, int dividerSize,
            boolean isHorizontalDivision, Rect insets, int dockSide, boolean isMinimizedMode,
            boolean isHomeResizable) {
            mSnapMode = isMinimizedMode ? SNAP_MODE_MINIMIZED :
                res.getInteger(com.android.internal.R.integer.config_dockedStackDividerSnapMode);
  }
//res/config.xml
<integer name="config_dockedStackDividerSnapMode">0</integer>
//res-land/config.xml
<integer name="config_dockedStackDividerSnapMode">2</integer>  
    
 	/**
     * 3 snap targets: left/top has 16:9 ratio (for videos), 1:1, and right/bottom has 16:9 ratio
     */
    private static final int SNAP_MODE_16_9 = 0;

    /**
     * 3 snap targets: fixed ratio, 1:1, (1 - fixed ratio)
     */
    private static final int SNAP_FIXED_RATIO = 1;

    /**
     * 1 snap target: 1:1
     */
    private static final int SNAP_ONLY_1_1 = 2;
```

竖屏值一般是SNAP_MODE_16_9=0，横屏值一般是SNAP_ONLY_1_1=2

竖屏的3个snap的计算方法addRatio16_9Targets

```java
private void addRatio16_9Targets(boolean isHorizontalDivision, int dividerMax) {
    int start = isHorizontalDivision ? mInsets.top : mInsets.left;
    int end = isHorizontalDivision
            ? mDisplayHeight - mInsets.bottom
            : mDisplayWidth - mInsets.right;
    int startOther = isHorizontalDivision ? mInsets.left : mInsets.top;
    int endOther = isHorizontalDivision
            ? mDisplayWidth - mInsets.right
            : mDisplayHeight - mInsets.bottom;
    float size = 9.0f / 16.0f * (endOther - startOther);
    int sizeInt = (int) Math.floor(size);
    int topPosition = start + sizeInt;
    int bottomPosition = end - sizeInt - mDividerSize;
    //计算了topPosition，bottomPosition两个位置，就是3分割线的上下分割线位置
    addNonDismissingTargets(isHorizontalDivision, topPosition, bottomPosition, dividerMax);
}

  private void addNonDismissingTargets(boolean isHorizontalDivision, int topPosition,
            int bottomPosition, int dividerMax) {
        maybeAddTarget(topPosition, topPosition - getStartInset());
        addMiddleTarget(isHorizontalDivision);
        maybeAddTarget(bottomPosition,
                dividerMax - getEndInset() - (bottomPosition + mDividerSize));
    }

    private void addMiddleTarget(boolean isHorizontalDivision) {
        int position = DockedDividerUtils.calculateMiddlePosition(isHorizontalDivision,
                mInsets, mDisplayWidth, mDisplayHeight, mDividerSize);
        mTargets.add(new SnapTarget(position, position, SnapTarget.FLAG_NONE));
    }
```

addNonDismissingTargets方法主要就是有了topPosition，bottomPosition，在额外加上个middle，这样整体3个SnapTarget就构造成功。创建后的分割线SnapTarget都会被放入到mTargets这个集合中去

SnapTarget对应中间位置的堆栈

```
at com.android.internal.policy.DividerSnapAlgorithm$SnapTarget.<init>(DividerSnapAlgorithm.java:472)
at com.android.internal.policy.DividerSnapAlgorithm$SnapTarget.<init>(DividerSnapAlgorithm.java:468)
at com.android.internal.policy.DividerSnapAlgorithm.maybeAddTarget(DividerSnapAlgorithm.java:356)
at com.android.internal.policy.DividerSnapAlgorithm.addNonDismissingTargets(DividerSnapAlgorithm.java:317)
at com.android.internal.policy.DividerSnapAlgorithm.addRatio16_9Targets(DividerSnapAlgorithm.java:347)
at com.android.internal.policy.DividerSnapAlgorithm.calculateTargets(DividerSnapAlgorithm.java:299)
at com.android.internal.policy.DividerSnapAlgorithm.<init>(DividerSnapAlgorithm.java:138)
at com.android.internal.policy.DividerSnapAlgorithm.<init>(DividerSnapAlgorithm.java:112)
at com.android.wm.shell.common.split.SplitLayout.getSnapAlgorithm(SplitLayout.java:433)
at com.android.wm.shell.common.split.SplitLayout.<init>(SplitLayout.java:134)
at com.android.wm.shell.splitscreen.StageCoordinator.onTaskAppeared(StageCoordinator.java:958)
at com.android.wm.shell.ShellTaskOrganizer.onTaskAppeared(ShellTaskOrganizer.java:438)
at com.android.wm.shell.ShellTaskOrganizer.onTaskAppeared(ShellTaskOrganizer.java:427)
at android.window.TaskOrganizer$1.lambda$onTaskAppeared$4$android-window-TaskOrganizer$1(TaskOrganizer.java:306)
```

##### 寻找合适分割线

**外部传递一个比例值情况**
一般这种情况是在启动分屏时候，需要设置好一个上下分屏比例ratio,主要通过如下方法：

```java
/** Updates divide position and split bounds base on the ratio within root bounds. */
public void setDivideRatio(float ratio) {
    final int position = isLandscape()
            ? mRootBounds.left + (int) (mRootBounds.width() * ratio)
            : mRootBounds.top + (int) (mRootBounds.height() * ratio);
    final DividerSnapAlgorithm.SnapTarget snapTarget =
            mDividerSnapAlgorithm.calculateNonDismissingSnapTarget(position);
    setDividePosition(snapTarget.position, false /* applyLayoutChange */);
}

   public SnapTarget calculateNonDismissingSnapTarget(int position) {
        SnapTarget target = snap(position, false /* hardDismiss */);//主要调用snap
        if (target == mDismissStartTarget) {
            return mFirstSplitTarget;
        } else if (target == mDismissEndTarget) {
            return mLastSplitTarget;
        } else {
            return target;
        }
    }

    private SnapTarget snap(int position, boolean hardDismiss) {
        if (shouldApplyFreeSnapMode(position)) {
            return new SnapTarget(position, position, SnapTarget.FLAG_NONE);
        }
        int minIndex = -1;
        float minDistance = Float.MAX_VALUE;
        int size = mTargets.size();
        for (int i = 0; i < size; i++) {
            SnapTarget target = mTargets.get(i);
            float distance = Math.abs(position - target.position);
            if (hardDismiss) {
                distance /= target.distanceMultiplier;
            }
            if (distance < minDistance) {//这里会计算传递进来position和SnapTarget.position的距离
                minIndex = i;
                minDistance = distance;
            }
        }
        return mTargets.get(minIndex);
    }
```

snap方法根据传递进来的position，与mTargets集合的所有position进行比较距离，离谁最近就选谁。

##### 分割线拖动

在DividerView的onTouch方法中

```java
    public boolean onTouch(View v, MotionEvent event) {
        if (mSplitLayout == null || !mInteractive) {
            return false;
        }

        if (mDoubleTapDetector.onTouchEvent(event)) {
            return true;
        }

        // Convert to use screen-based coordinates to prevent lost track of motion events while
        // moving divider bar and calculating dragging velocity.
        event.setLocation(event.getRawX(), event.getRawY());
        final int action = event.getAction() & MotionEvent.ACTION_MASK;
        final boolean isLandscape = isLandscape();
        final int touchPos = (int) (isLandscape ? event.getX() : event.getY());
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mVelocityTracker = VelocityTracker.obtain();
                mVelocityTracker.addMovement(event);
                setTouching();
                mStartPos = touchPos;
                mMoving = false;
                break;
            case MotionEvent.ACTION_MOVE:
                mVelocityTracker.addMovement(event);
                if (!mMoving && Math.abs(touchPos - mStartPos) > mTouchSlop) {
                    mStartPos = touchPos;
                    mMoving = true;
                }
                if (mMoving) {
                    final int position = mSplitLayout.getDividePosition() + touchPos - mStartPos;
                    mSplitLayout.updateDivideBounds(position);//最重要的拖动方法
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                releaseTouching();
                if (!mMoving) break;

                mVelocityTracker.addMovement(event);
                mVelocityTracker.computeCurrentVelocity(1000 /* units */);
                final float velocity = isLandscape
                        ? mVelocityTracker.getXVelocity()
                        : mVelocityTracker.getYVelocity();
                final int position = mSplitLayout.getDividePosition() + touchPos - mStartPos;
                final DividerSnapAlgorithm.SnapTarget snapTarget =
                        mSplitLayout.findSnapTarget(position, velocity, false /* hardDismiss */);
                mSplitLayout.snapToTarget(position, snapTarget);//松手后要进行动画，DividerView自动到某个区域
                mMoving = false;
                break;
        }
        return true;
    }


```

重点看下mSplitLayout.updateDivideBounds(position)方法

```java
    /**
     * Updates bounds with the passing position. Usually used to update recording bounds while
     * performing animation or dragging divider bar to resize the splits.
     */
    void updateDivideBounds(int position) {
        updateBounds(position);//更新一下bound区域
        mSplitLayoutHandler.onLayoutSizeChanging(this);
        //会调用对应的onLayoutSizeChanging方法来负责通知size变化了
    }

   @Override
    public void onLayoutSizeChanging(SplitLayout layout) {
        final SurfaceControl.Transaction t = mTransactionPool.acquire();
        t.setFrameTimelineVsync(Choreographer.getInstance().getVsyncId());
        setResizingSplits(true /* resizing */);
        //会把对应的DividerView进行坐标更新，实现跟手
        updateSurfaceBounds(layout, t, true /* applyResizingOffset */);
        //主区域有size变化，如果bounds变大则显示icon+背景，否则显示原来内容，并控制内容的区域跟随divideriew位置
        mMainStage.onResizing(getMainStageBounds(), t);
        //次区域有size变化，如果bounds变大则显示icon+背景，否则显示原来内容，并控制内容的区域跟随divideriew位置
        mSideStage.onResizing(getSideStageBounds(), t);
        t.apply();
        mTransactionPool.release(t);
    }
       void updateSurfaceBounds(@Nullable SplitLayout layout, @NonNull SurfaceControl.Transaction t,
            boolean applyResizingOffset) {
        final StageTaskListener topLeftStage =
                mSideStagePosition == SPLIT_POSITION_TOP_OR_LEFT ? mSideStage : mMainStage;
        final StageTaskListener bottomRightStage =
                mSideStagePosition == SPLIT_POSITION_TOP_OR_LEFT ? mMainStage : mSideStage;
         //这里关键调用了applySurfaceChanges
        (layout != null ? layout : mSplitLayout).applySurfaceChanges(t, topLeftStage.mRootLeash,
                bottomRightStage.mRootLeash, topLeftStage.mDimLayer, bottomRightStage.mDimLayer,
                applyResizingOffset);
    }

```

applySurfaceChanges方法

```java
public void applySurfaceChanges(SurfaceControl.Transaction t, SurfaceControl leash1,
            SurfaceControl leash2, SurfaceControl dimLayer1, SurfaceControl dimLayer2,
            boolean applyResizingOffset) {
        final SurfaceControl dividerLeash = getDividerLeash();
        if (dividerLeash != null) {
            mTempRect.set(getRefDividerBounds());
            t.setPosition(dividerLeash, mTempRect.left, mTempRect.top);
            //这里是设置DeviderView进行位置设置
            // Resets layer of divider bar to make sure it is always on top.
            t.setLayer(dividerLeash, Integer.MAX_VALUE);
        }
        mTempRect.set(getRefBounds1());
    	//设置上下分屏的位置和裁减
        t.setPosition(leash1, mTempRect.left, mTempRect.top)
                .setWindowCrop(leash1, mTempRect.width(), mTempRect.height());
        mTempRect.set(getRefBounds2());
        t.setPosition(leash2, mTempRect.left, mTempRect.top)
                .setWindowCrop(leash2, mTempRect.width(), mTempRect.height());

        if (mImePositionProcessor.adjustSurfaceLayoutForIme(
                t, dividerLeash, leash1, leash2, dimLayer1, dimLayer2)) {
            return;
        }
		//进行对应的特效显示，拉到底部或顶部进行灰度提示
        mSurfaceEffectPolicy.adjustDimSurface(t, dimLayer1, dimLayer2);
        if (applyResizingOffset) {
            mSurfaceEffectPolicy.adjustRootSurface(t, leash1, leash2);
        }
    }
```

mMainStage.onResizing方法

```java

 /** Showing resizing hint. */
    public void onResizing(ActivityManager.RunningTaskInfo resizingTask, Rect newBounds,
            SurfaceControl.Transaction t) {
   //这里本质就是判断是否该区域有变大，如果有变大则变成ICON+背景，另一个屏则保持原样，因为show是为false
        final boolean show =
                newBounds.width() > mBounds.width() || newBounds.height() > mBounds.height();
 		...
        t.setPosition(mIconLeash,
                newBounds.width() / 2 - mIconSize / 2,
                newBounds.height() / 2 - mIconSize / 2);

        if (animate) {//启动相关显示的动画
            startFadeAnimation(show, false /* isResized */);
            mShown = show;
        }
    }
```

松手后mSplitLayout.snapToTarget(position, snapTarget)执行

```java
  /**
     * Sets new divide position and updates bounds correspondingly. Notifies listener if the new
     * target indicates dismissing split.
     */
    public void snapToTarget(int currentPosition, DividerSnapAlgorithm.SnapTarget snapTarget) {
        switch (snapTarget.flag) {
            //DividerView上滑到顶部，导致上分屏退出
            case FLAG_DISMISS_START:
                flingDividePosition(currentPosition, snapTarget.position,
                        () -> mSplitLayoutHandler.onSnappedToDismiss(false /* bottomOrRight */));
                break;
            case FLAG_DISMISS_END:
                flingDividePosition(currentPosition, snapTarget.position,
                        () -> mSplitLayoutHandler.onSnappedToDismiss(true /* bottomOrRight */));
                break;
            default:
                flingDividePosition(currentPosition, snapTarget.position,
                        () -> setDividePosition(snapTarget.position, true /* applyLayoutChange */));
                break;
        }
    }
  
    void flingDividePosition(int from, int to, @Nullable Runnable flingFinishedCallback) {
        if (from == to) {
            // No animation run, still callback to stop resizing.
            mSplitLayoutHandler.onLayoutSizeChanged(this);
            return;
        }
        InteractionJankMonitorUtils.beginTracing(InteractionJankMonitor.CUJ_SPLIT_SCREEN_RESIZE,
                mSplitWindowManager.getDividerView(), "Divider fling");
        ValueAnimator animator = ValueAnimator
                .ofInt(from, to)
                .setDuration(250);//开始设置动画，一个from一个to作为开始和截至地方
        animator.setInterpolator(Interpolators.FAST_OUT_SLOW_IN);
        animator.addUpdateListener(
                animation -> updateDivideBounds((int) animation.getAnimatedValue()));
        animator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
            	//动画结束以后的最重要的操作，需要真正的windowcontainer的bounds进行操作了
                if (flingFinishedCallback != null) {
                    flingFinishedCallback.run();//执行传递进来的那个runable
                }
                InteractionJankMonitorUtils.endTracing(
                        InteractionJankMonitor.CUJ_SPLIT_SCREEN_RESIZE);
            }

            @Override
            public void onAnimationCancel(Animator animation) {
                setDividePosition(to, true /* applyLayoutChange */);
            }
        });
        animator.start();
    }
```

#### 分屏退出

##### SystemUI端的调用

```java
//frameworks/base/libs/WindowManager/Shell/src/com/android/wm/shell/splitscreen/StageCoordinator.java
private void applyExitSplitScreen(@Nullable StageTaskListener childrenToTop,
            WindowContainerTransaction wct, @ExitReason int exitReason) {
        final boolean fromEnteringPip = exitReason == EXIT_REASON_CHILD_TASK_ENTER_PIP;
     	//移除对应的sidestate下面task节点,其实本质就是把子节点进行对应的reparent到新的父亲
        mSideStage.removeAllTasks(wct, !fromEnteringPip && mSideStage == childrenToTop);
     	//和上面的removeAllTask类似，都是对task进行reparent操作，不能再继续挂载在分屏的节点了
        mMainStage.deactivate(wct, !fromEnteringPip && mMainStage == childrenToTop);
        
        //对分屏的总节点进行对应的reorder，放到后面了
        wct.reorder(mRootTaskInfo.token, false /* onTop */);
        mTaskOrganizer.applyTransaction(wct);
        mSyncQueue.runInSync(t -> {
            setResizingSplits(false /* resizing */);
            t.setWindowCrop(mMainStage.mRootLeash, null)
                    .setWindowCrop(mSideStage.mRootLeash, null);
            setDividerVisibility(false, t);
        });
    }
```

mSideStage.removeAllTasks方法

```java
   boolean removeAllTasks(WindowContainerTransaction wct, boolean toTop) {
        if (mChildrenTaskInfo.size() == 0) return false;
        wct.reparentTasks(
                mRootTaskInfo.token,
                null /* newParent */,
                CONTROLLED_WINDOWING_MODES_WHEN_ACTIVE,
                CONTROLLED_ACTIVITY_TYPES,
                toTop);
        return true;
    }
```

调用到了reparentTasks

```java
public WindowContainerTransaction reparentTasks(@Nullable WindowContainerToken currentParent,
            @Nullable WindowContainerToken newParent, @Nullable int[] windowingModes,
            @Nullable int[] activityTypes, boolean onTop, boolean reparentTopOnly) {
        mHierarchyOps.add(HierarchyOp.createForChildrenTasksReparent(
                currentParent != null ? currentParent.asBinder() : null,
                newParent != null ? newParent.asBinder() : null,
                windowingModes,
                activityTypes,
                onTop,
                reparentTopOnly));
        return this;
    }

		//创建对应HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT的HierarchyOp
        public static HierarchyOp createForChildrenTasksReparent(IBinder currentParent,
                IBinder newParent, int[] windowingModes, int[] activityTypes, boolean onTop,
                boolean reparentTopOnly) {
            return new HierarchyOp.Builder(HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT)
                    .setContainer(currentParent)
                    .setReparentContainer(newParent)
                    .setWindowingModes(windowingModes)
                    .setActivityTypes(activityTypes)
                    .setToTop(onTop)
                    .setReparentTopOnly(reparentTopOnly)
                    .build();
        }

```

##### systemserver端的调用

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java
private int applyHierarchyOp(WindowContainerTransaction.HierarchyOp hop, int effects,
            int syncId, @Nullable Transition transition, boolean isInLockTaskMode,
            @NonNull CallerInfo caller, @Nullable IBinder errorCallbackToken,
            @Nullable ITaskFragmentOrganizer organizer, @Nullable Transition finishTransition) {
            ...
            switch (type) {
            case HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT: {
            //核心方法
                effects |= reparentChildrenTasksHierarchyOp(hop, transition, syncId);
                break;
            }
           	...
            }   

 private int reparentChildrenTasksHierarchyOp(WindowContainerTransaction.HierarchyOp hop,
            @Nullable Transition transition, int syncId) {
        WindowContainer<?> currentParent = hop.getContainer() != null
                ? WindowContainer.fromBinder(hop.getContainer()) : null;
        WindowContainer newParent = hop.getNewParent() != null
                ? WindowContainer.fromBinder(hop.getNewParent()) : null;
        if (currentParent == null && newParent == null) {
          ...
        } else if (newParent == null) {//这里传递为null，所以给予没人的parent为DefaultTaskDisplayArea
            newParent = currentParent.asTask().getDisplayContent().getDefaultTaskDisplayArea();
        }
       ...
        final ArrayList<Task> tasksToReparent = new ArrayList<>();
		//对当前的task节点下面所有children进行遍历
        currentParent.forAllTasks(task -> {
            final boolean reparent;
            //可能activitytype和windowmode不支持，就要直接返回不会放入tasksToReparent
            if (!ArrayUtils.contains(hop.getActivityTypes(), task.getActivityType())
                    || !ArrayUtils.contains(hop.getWindowingModes(), task.getWindowingMode())) {
                return false;
            }
			//如果放到顶部的就放到
            if (hop.getToTop()) {
                tasksToReparent.add(0, task);
            } else {
                tasksToReparent.add(task);
            }
            return hop.getReparentTopOnly() && tasksToReparent.size() == 1;
        });

        final int count = tasksToReparent.size();
        for (int i = 0; i < count; ++i) {
            final Task task = tasksToReparent.get(i);
          ...
          	//newParent是TaskDisplayArea，即直接把原理挂在上下分屏的task挂到TaskDisplayArea下面
            if (newParent instanceof TaskDisplayArea) {
                // For now, reparenting to display area is different from other reparents...
                task.reparent((TaskDisplayArea) newParent, hop.getToTop());
            } else {
                task.reparent((Task) newParent,
                        hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM,
                        false /*moveParents*/, "processChildrenTaskReparentHierarchyOp");
            }
        }
        return TRANSACT_EFFECTS_LIFECYCLE;
    }
```

整个分屏结束核心流程结束，剩下的就是要进行相关resumeToActivity和ensureVisibleActivity等操作，触发生命周期的变化。
