---

title: Android 13 freeform自由窗口
date: 2023-9-21 22:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

#### 自由窗口打开配置

aosp默认没有开启自由窗口模式，可以使用如下命令开启

```
adb shell settings put global enable_freeform_support 1
adb shell settings put global force_resizable_activities 1
```

或者在开发者选项中打开

- 强制将Activity设置为可调整大小
- 启用可自由调整的窗口

开启之后可以在最近任务栏窗口应用图标选择【自由窗口】打开自由窗口应用。

#### 进入freeform分析

##### 上层启动部分

第一步点击按钮开启的堆栈

```java
com.android.systemui.shared.system.ActivityManagerWrapper.startActivityFromRecents(ActivityManagerWrapper.java:279)
com.android.quickstep.TaskShortcutFactory$MultiWindowSystemShortcut.onClick(TaskShortcutFactory.java:196)
com.android.quickstep.views.TaskMenuView.lambda$addMenuOption$1(TaskMenuView.java:254)
com.android.quickstep.views.TaskMenuView$$ExternalSyntheticLambda2.run(Unknown Source:4)
com.android.quickstep.views.RecentsView.finishRecentsAnimation(RecentsView.java:4536)
com.android.quickstep.views.TaskMenuView.lambda$addMenuOption$2(TaskMenuView.java:252)
com.android.quickstep.views.TaskMenuView$$ExternalSyntheticLambda1.run(Unknown Source:6)
com.android.quickstep.views.RecentsView.switchToScreenshot(RecentsView.java:4962)
com.android.quickstep.views.TaskMenuView.lambda$addMenuOption$3$com-android-quickstep-views-TaskMenuView(TaskMenuView.java:251)
com.android.quickstep.views.TaskMenuView$$ExternalSyntheticLambda5.onClick(Unknown Source:4)
```

MultiWindowSystemShortcut.onClick对应代码

```java
//packages/apps/Launcher3/quickstep/src/com/android/quickstep/TaskShortcutFactory.java
TaskShortcutFactory FREE_FORM = new MultiWindowFactory(R.drawable.ic_split_screen,
            R.string.recent_task_option_freeform, LAUNCHER_SYSTEM_SHORTCUT_FREE_FORM_TAP) {
    @Override
        protected ActivityOptions makeLaunchOptions(Activity activity) {
            //设置setLaunchWindowingMode(WINDOWING_MODE_FREEFORM);分屏模式
            ActivityOptions activityOptions = ActivityOptionsCompat.makeFreeformOptions();
            // Arbitrary bounds only because freeform is in dev mode right now
            Rect r = new Rect(50, 50, 200, 200);
            activityOptions.setLaunchBounds(r);
            return activityOptions;
        }
}
class MultiWindowSystemShortcut extends SystemShortcut<BaseDraggingActivity> {
        @Override
        public void onClick(View view) {
            Task.TaskKey taskKey = mTaskView.getTask().key;
            final int taskId = taskKey.id;
            ActivityOptions options = mFactory.makeLaunchOptions(mTarget);
            if (options != null) {
                options.setSplashScreenStyle(SplashScreen.SPLASH_SCREEN_STYLE_ICON);
            }
            if (options != null
                    && ActivityManagerWrapper.getInstance().startActivityFromRecents(taskId,
                            options)) {
    }
  /**
     * Starts a task from Recents synchronously.
     */
    public boolean startActivityFromRecents(int taskId, ActivityOptions options) {
        try {
            Bundle optsBundle = options == null ? null : options.toBundle();
            getService().startActivityFromRecents(taskId, optsBundle);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
            
```

##### SystemServer部分

点击时候会触发startActivityFromRecents，会跨进程调到ActivityTaskManagerService

```
com.android.server.wm.Task.moveToFrontInner(Task.java:4674)
com.android.server.wm.Task.moveToFront(Task.java:4662)
com.android.server.wm.ActivityRecord.moveFocusableActivityToTop(ActivityRecord.java:3240)
com.android.server.wm.Task.moveTaskToFront(Task.java:5557)
com.android.server.wm.Task.moveTaskToFront(Task.java:5511)
com.android.server.wm.ActivityTaskSupervisor.findTaskToMoveToFront(ActivityTaskSupervisor.java:1474)
com.android.server.wm.ActivityTaskManagerService.moveTaskToFrontLocked(ActivityTaskManagerService.java:2158)
com.android.server.wm.ActivityTaskSupervisor.startActivityFromRecents(ActivityTaskSupervisor.java:2575)
com.android.server.wm.ActivityTaskManagerService.startActivityFromRecents(ActivityTaskManagerService.java:1747)
```

接下来重下ActivityTaskSupervisor.startActivityFromRecents

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java
 /**
     * Start the given task from the recent tasks. Do not hold WM global lock when calling this
     * method to avoid potential deadlock or permission deny by UriGrantsManager when resolving
     * activity (see {@link ActivityStarter.Request#resolveActivity} and
     * {@link com.android.server.am.ContentProviderHelper#checkContentProviderUriPermission}).
     *
     * @return The result code of starter.
     */
    int startActivityFromRecents(int callingPid, int callingUid, int taskId,
            SafeActivityOptions options) {
        synchronized (mService.mGlobalLock) {
            int activityType = ACTIVITY_TYPE_UNDEFINED;
            if (activityOptions != null) {
                activityType = activityOptions.getLaunchActivityType();
                final int windowingMode = activityOptions.getLaunchWindowingMode();
            try {
                //根据taskId获取Task对象
                task = mRootWindowContainer.anyTaskForId(taskId,
                        MATCH_ATTACHED_TASK_OR_RECENT_TASKS_AND_RESTORE, activityOptions, ON_TOP);                           // If the user must confirm credentials (e.g. when first launching a work
                // app and the Work Challenge is present) let startActivityInPackage handle
                // the intercepting.
                if (!mService.mAmInternal.shouldConfirmCredentials(task.mUserId)
                        && task.getRootActivity() != null) {             
                    try {
                    	//把这个Freeform的Task移到最顶部前端
                        mService.moveTaskToFrontLocked(null /* appThread */,
                                null /* callingPackage */, task.mTaskId, 0, options);
                        // Apply options to prevent pendingOptions be taken when scheduling
                        // activity lifecycle transaction to make sure the override pending app
                        // transition will be applied immediately.
                        targetActivity.applyOptionsAnimation();
                    } finally {
                        mActivityMetricsLogger.notifyActivityLaunched(launchingState,
                                START_TASK_TO_FRONT, false /* newActivityCreated */,
                                targetActivity, activityOptions);
                    }

                    mService.getActivityStartController().postStartActivityProcessingForLastStarter(
                            task.getTopNonFinishingActivity(), ActivityManager.START_TASK_TO_FRONT,
                            task.getRootTask());

                    // As it doesn't go to ActivityStarter.executeRequest() path, we need to resume
                    // app switching here also.
                    mService.resumeAppSwitches();
                    return ActivityManager.START_TASK_TO_FRONT;
                }
    }
```

findTaskToMoveToFront方法

```java

 /** This doesn't just find a task, it also moves the task to front. */
    void findTaskToMoveToFront(Task task, int flags, ActivityOptions options, String reason,
            boolean forceNonResizeable) {
        Task currentRootTask = task.getRootTask();
            if (task.isResizeable() && canUseActivityOptionsLaunchBounds(options)) {
                final Rect bounds = options.getLaunchBounds();
                // 直接给task设置bounds
                task.setBounds(bounds);

                Task targetRootTask =
                        mRootWindowContainer.getOrCreateRootTask(null, options, task, ON_TOP);
                 //这里判断一般针对pip那种
                if (targetRootTask.shouldResizeRootTaskWithLaunchBounds()) {
                    targetRootTask.resize(bounds, !PRESERVE_WINDOWS, !DEFER_RESUME);
                } else {
                    // WM resizeTask must be done after the task is moved to the correct stack,
                    // because Task's setBounds() also updates dim layer's bounds, but that has
                    // dependency on the root task.
                    //调用task的resize
                    task.resize(false /* relayout */, false /* forced */);
                }
            }
            final ActivityRecord r = task.getTopNonFinishingActivity();
            //把task真正移到前台
            currentRootTask.moveTaskToFront(task, false /* noAnimation */, options,
                    r == null ? null : r.appTimeTracker, reason);
        } finally {
            mUserLeaving = false;
        }
    }
```

##### 显示大小

```java
TaskShortcutFactory FREE_FORM = new MultiWindowFactory(R.drawable.ic_split_screen,
            R.string.recent_task_option_freeform, LAUNCHER_SYSTEM_SHORTCUT_FREE_FORM_TAP) {
    @Override
        protected ActivityOptions makeLaunchOptions(Activity activity) {
            //设置setLaunchWindowingMode(WINDOWING_MODE_FREEFORM);分屏模式
            ActivityOptions activityOptions = ActivityOptionsCompat.makeFreeformOptions();
            // Arbitrary bounds only because freeform is in dev mode right now
            Rect r = new Rect(50, 50, 200, 200);
            activityOptions.setLaunchBounds(r);
            return activityOptions;
        }
}
```

显示大小设置的50, 50, 200, 200，但是事件显示的大小不是这个大小

```
adb shell am stack list
RootTask id=227 bounds=[50,84][470,504] displayId=0 userId=0
  taskId=227: com.android.messaging/com.android.messaging.ui.conversationlist.ConversationListActivity bounds=[50,84][470,504]
```

setBounds堆栈打印日志

```
android.app.WindowConfiguration.setBounds(WindowConfiguration.java:289)
android.app.WindowConfiguration.setTo(WindowConfiguration.java:451)
android.content.res.Configuration.setTo(Configuration.java:1047)
com.android.server.wm.ConfigurationContainer.resolveOverrideConfiguration(ConfigurationContainer.java:171)
com.android.server.wm.TaskFragment.resolveOverrideConfiguration(TaskFragment.java:1891)
com.android.server.wm.ConfigurationContainer.onConfigurationChanged(ConfigurationContainer.java:127)
com.android.server.wm.WindowContainer.onConfigurationChanged(WindowContainer.java:510)
com.android.server.wm.TaskFragment.onConfigurationChanged(TaskFragment.java:2261)
com.android.server.wm.Task.onConfigurationChangedInner(Task.java:1903)
com.android.server.wm.Task.onConfigurationChanged(Task.java:1988)
com.android.server.wm.ConfigurationContainer.onRequestedOverrideConfigurationChanged(ConfigurationContainer.java:2
com.android.server.wm.WindowContainer.onRequestedOverrideConfigurationChanged(WindowContainer.java:979)
com.android.server.wm.ConfigurationContainer.setBounds(ConfigurationContainer.java:375)
com.android.server.wm.Task.setBounds(Task.java:5915)
com.android.server.wm.Task.setBounds(Task.java:2617)
com.android.server.wm.ActivityTaskSupervisor.findTaskToMoveToFront(ActivityTaskSupervisor.java:1445)
com.android.server.wm.ActivityTaskManagerService.moveTaskToFrontLocked(ActivityTaskManagerService.java:2158)
com.android.server.wm.ActivityTaskSupervisor.startActivityFromRecents(ActivityTaskSupervisor.java:2575)
com.android.server.wm.ActivityTaskManagerService.startActivityFromRecents(ActivityTaskManagerService.java:1747)
```

```java
//TaskFragment.resolveOverrideConfiguration
void resolveOverrideConfiguration(Configuration newParentConfig) {
        mTmpBounds.set(getResolvedOverrideConfiguration().windowConfiguration.getBounds());
        super.resolveOverrideConfiguration(newParentConfig);
        final Task thisTask = asTask();
        // Embedded Task's configuration should go with parent TaskFragment, so we don't re-compute
        // configuration here.
        if (thisTask != null && !thisTask.isEmbedded()) {
            //对task的override的Config进行校验从新设定
            thisTask.resolveLeafTaskOnlyOverrideConfigs(newParentConfig,
                    mTmpBounds /* previousBounds */);
        }
        computeConfigResourceOverrides(getResolvedOverrideConfiguration(), newParentConfig);
    }

     void resolveLeafTaskOnlyOverrideConfigs(Configuration newParentConfig, Rect previousBounds) {
        //这里会进行相关的最小尺寸适配，这里就把窗口的坐标变大
        adjustForMinimalTaskDimensions(outOverrideBounds, previousBounds, newParentConfig);
        if (windowingMode == WINDOWING_MODE_FREEFORM) {
            computeFreeformBounds(outOverrideBounds, newParentConfig);
            return;
        }
    }

     void adjustForMinimalTaskDimensions(@NonNull Rect bounds, @NonNull Rect previousBounds,
            @NonNull Configuration parentConfig) {
        int minWidth = mMinWidth;
        int minHeight = mMinHeight;
        // If the task has no requested minimal size, we'd like to enforce a minimal size
        // so that the user can not render the task fragment too small to manipulate. We don't need
        // to do this for the root pinned task as the bounds are controlled by the system.
        if (!inPinnedWindowingMode()) {
            // Use Display specific min sizes when there is one associated with this Task.
            final int defaultMinSizeDp = mDisplayContent == null
                    ? DEFAULT_MIN_TASK_SIZE_DP : mDisplayContent.mMinSizeOfResizeableTaskDp;
            //这里就相当于wms中获取了mMinSizeOfResizeableTaskDp最小坐标，220dp默认
            final float density = (float) parentConfig.densityDpi / DisplayMetrics.DENSITY_DEFAULT;
            final int defaultMinSize = (int) (defaultMinSizeDp * density);
            if (minWidth == INVALID_MIN_SIZE) {
                minWidth = defaultMinSize;
            }
            if (minHeight == INVALID_MIN_SIZE) {
                minHeight = defaultMinSize;
            }
        }     
    }
    
 /** Computes bounds for {@link WindowConfiguration#WINDOWING_MODE_FREEFORM}. */
    private void computeFreeformBounds(@NonNull Rect outBounds,
            @NonNull Configuration newParentConfig) {
        // by policy, make sure the window remains within parent somewhere
        final float density =
                ((float) newParentConfig.densityDpi) / DisplayMetrics.DENSITY_DEFAULT;
        final Rect parentBounds =
                new Rect(newParentConfig.windowConfiguration.getBounds());
        final DisplayContent display = getDisplayContent();
        if (display != null) {
            // If a freeform window moves below system bar, there is no way to move it again
            // by touch. Because its caption is covered by system bar. So we exclude them
            // from root task bounds. and then caption will be shown inside stable area.
            final Rect stableBounds = new Rect();
            display.getStableRect(stableBounds);
            parentBounds.intersect(stableBounds);
        }

        fitWithinBounds(outBounds, parentBounds,
                (int) (density * WindowState.MINIMUM_VISIBLE_WIDTH_IN_DP),
                (int) (density * WindowState.MINIMUM_VISIBLE_HEIGHT_IN_DP));

        // Prevent to overlap caption with stable insets.
        //对top的区域进行修正
         int offsetTop = parentBounds.top - outBounds.top;
        if (offsetTop > 0) {
            outBounds.offset(0, offsetTop);
        }
    }
```

50, 50, 200, 200最终变成50,84，470,504，原因是

density=3.5，default_minimal_size_resizable_task设置了120，最小宽高是120*3.5=420

原数据宽度150，不满足最小，宽度最小420，200就会变成470

顶部会变成状态栏的高度84，高度84+420 = 504

计算下，这些数据就能对的上了。

#### DecorCaptionView

对应打开之后顶部显示的一些不属于Activity的按钮区域，在DecorView增加的一个DecorCaptionView，可以控制窗口。

```
  View Hierarchy:
      DecorView@1c1e1e8[ConversationListActivity]
        com.android.internal.widget.DecorCaptionView{283a94e V.E...... ........ 0,0-420,420}
          android.widget.LinearLayout{ff90a6f V.E...... ........ 0,0-420,420}
```

打开对应的堆栈

```java
com.android.internal.policy.DecorView.createDecorCaptionView(DecorView.java:2248)
com.android.internal.policy.DecorView.onResourcesLoaded(DecorView.java:2207)
com.android.internal.policy.PhoneWindow.generateLayout(PhoneWindow.java:2674)
com.android.internal.policy.PhoneWindow.installDecor(PhoneWindow.java:2737)
com.android.internal.policy.PhoneWindow.getDecorView(PhoneWindow.java:2144)
androidx.appcompat.app.AppCompatActivity.initViewTreeOwners(AppCompatActivity.java:219)
androidx.appcompat.app.AppCompatActivity.setContentView(AppCompatActivity.java:194)
com.android.messaging.ui.conversationlist.ConversationListActivity.onCreate(ConversationListActivity.java:35)
android.app.Activity.performCreate(Activity.java:8290)
android.app.Activity.performCreate(Activity.java:8269)
android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1384)
android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3657)
android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3813)
android.app.ActivityThread.handleRelaunchActivityInner(ActivityThread.java:5791)
android.app.ActivityThread.handleRelaunchActivity(ActivityThread.java:5682)
android.app.servertransaction.ActivityRelaunchItem.execute(ActivityRelaunchItem.java:71)
```

handleRelaunchActivity时候进行的创建，systemserver端触发的操作

```
com.android.server.wm.ActivityRecord.relaunchActivityLocked
com.android.server.wm.TaskFragment.resumeTopActivity
com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities
com.android.server.wm.Task.moveTaskToFront
```

#### 自由窗口的移动

自由窗口的移到拖拽在顶部的CaptionView部分

应用部分

```java
//frameworks/base/core/java/com/android/internal/widget/DecorCaptionView.java
@Override
    public boolean onTouch(View v, MotionEvent e) {

            case MotionEvent.ACTION_MOVE:
					//app进程发起startMovingTask调用
                    startMovingTask(e.getRawX(), e.getRawY());
                break;


   public final boolean startMovingTask(float startX, float startY) {
        if (ViewDebug.DEBUG_POSITIONING) {
            Log.d(VIEW_LOG_TAG, "startMovingTask: {" + startX + "," + startY + "}");
        }
        try {
        	//通过Session进行跨进程调用到systemserver
            return mAttachInfo.mSession.startMovingTask(mAttachInfo.mWindow, startX, startY);
        } catch (RemoteException e) {
            Log.e(VIEW_LOG_TAG, "Unable to start moving", e);
        }
        return false;
    }

```

sytemServer部分

```java
 //frameworks/base/services/core/java/com/android/server/wm/Session.java
@Override
    public boolean startMovingTask(IWindow window, float startX, float startY) {
        final long ident = Binder.clearCallingIdentity();
        try {
       		//直接调用了TaskositioningController的startMovingTask
            return mService.mTaskPositioningController.startMovingTask(window, startX, startY);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
//frameworks/base/services/core/java/com/android/server/wm/TaskositioningController.java
	boolean startMovingTask(IWindow window, float startX, float startY) {
        WindowState win = null;
        synchronized (mService.mGlobalLock) {
            win = mService.windowForClientLocked(null, window, false);
            // win shouldn't be null here, pass it down to startPositioningLocked
            // to get warning if it's null.
            //调用了startPositioningLocked方法进行相关的触摸事件识别注册初始化
            if (!startPositioningLocked(
                    win, false /*resize*/, false /*preserveOrientation*/, startX, startY)) {
                return false;
            }
            mService.mAtmService.setFocusedTask(win.getTask().mTaskId);
        }
        return true;
    }
    
private boolean startPositioningLocked(WindowState win, boolean resize,
            boolean preserveOrientation, float startX, float startY) {
        mPositioningDisplay = displayContent;
		//进行触摸识别监听的TaskPositioner相关注册后如果有触摸事件产生则会调用到TaskPositioner的onInputEvent方法进行相关处理	
        mTaskPositioner = TaskPositioner.create(mService);
        mTaskPositioner.register(displayContent, win);
        return true;
    }

//TaskPositioner的onInputEvent方法进行相关触摸事件处理
private boolean onInputEvent(InputEvent event) {
        switch (motionEvent.getAction()) {
            case MotionEvent.ACTION_MOVE: {
            	//进行相关的move操作处理
                synchronized (mService.mGlobalLock) {
                	//根据坐标进行相关的bounds位置等计算
                    mDragEnded = notifyMoveLocked(newX, newY);
                    mTask.getDimBounds(mTmpRect);
                }
                if (!mTmpRect.equals(mWindowDragBounds)) {
                    //有了新计算的bounds进行task的对应resize操作
                    mService.mAtmService.resizeTask(
                            mTask.mTaskId, mWindowDragBounds, RESIZE_MODE_USER);
                }
            }
            break;
        }
        return true;
    }
```

#### 窗口大小变化

```java
//frameworks/base/services/core/java/com/android/server/wm/TaskTapPointerEventListener.java
 @Override
    public void onPointerEvent(MotionEvent motionEvent) {
        switch (motionEvent.getActionMasked()) {
            case MotionEvent.ACTION_DOWN: {
                final int x;
                final int y;
                if (motionEvent.getSource() == InputDevice.SOURCE_MOUSE) {
                    x = (int) motionEvent.getXCursorPosition();
                    y = (int) motionEvent.getYCursorPosition();
                } else {
                    x = (int) motionEvent.getX();
                    y = (int) motionEvent.getY();
                }

                synchronized (this) {
                    if (!mTouchExcludeRegion.contains(x, y)) {
                        //判定是否在区域内，不在区域内则执行handleTapOutsideTask
                        mService.mTaskPositioningController.handleTapOutsideTask(
                                mDisplayContent, x, y);
                    }
                }
            }
     //handleTapOutsideTask方法就会进行关键的区域判定是否触摸到了自由窗口task周围
	void handleTapOutsideTask(DisplayContent displayContent, int x, int y) {
        mService.mH.post(() -> {
            synchronized (mService.mGlobalLock) {
            	//根据x,y坐标进行Task的寻找
                final Task task = displayContent.findTaskForResizePoint(x, y);
                if (task != null) {
                    if (!task.isResizeable()) {
                        // The task is not resizable, so don't do anything when the user drags the
                        // the resize handles.
                        return;
                    }
                    //如果找到了task，那就开启相关的task拖拽识别，后面的逻辑就到了TaskPositioner中
                    if (!startPositioningLocked(task.getTopVisibleAppMainWindow(), true /*resize*/,
                            task.preserveOrientationOnResize(), x, y)) {
                        return;
                    }
                    mService.mAtmService.setFocusedTask(task.mTaskId);
                }
            }
        });
    }
   /**
     * Find the task whose outside touch area (for resizing) (x, y) falls within.
     * Returns null if the touch doesn't fall into a resizing area.
     */
    @Nullable
    Task findTaskForResizePoint(int x, int y) {
    	//task的触摸边界范围，即要触摸在delta范围以内，不然触摸就不起作用
        final int delta = dipToPixel(RESIZE_HANDLE_WIDTH_IN_DP, mDisplayMetrics);
        return getItemFromTaskDisplayAreas(taskDisplayArea ->
                mTmpTaskForResizePointSearchResult.process(taskDisplayArea, x, y, delta));
    }
```

原生自由窗口存在拖动时一直resize导致界面在刷新，然后外面拖动的区域很难发现。后面优化下拖动效果再添加下外边框。

#### 优化修改

##### 刷新闪烁问题修改

1、实现对Task固定屏幕宽，高度是全屏的一部分

2、实现对Task的SurfaceControl部分进行对应的缩放偏移

3、对freeform窗口放大缩小

部分修改关键代码：

```java
//TaskShortcutFactory.java
 protected ActivityOptions makeLaunchOptions(Activity activity) {
            //设置setLaunchWindowingMode(WINDOWING_MODE_FREEFORM);分屏模式
            ActivityOptions activityOptions = ActivityOptionsCompat.makeFreeformOptions();
            // Arbitrary bounds only because freeform is in dev mode right now
            //Rect r = new Rect(50, 50, 200, 200);
     		//固定Task大小
     		DisplayInfo info = DisplayInfo();
     		activity.getDisplay().getDisplayInfo(info);
     		Rect r = new Rect(0, 0, info.logicalWidth, (int)(info.logicalHeight * 63f));
            activityOptions.setLaunchBounds(r);
            return activityOptions;
        }
```

```java
//TaskFragment.java
Rect mFreeFormSurfaceBound =null;
//初始化位置和缩放
void initFreeformPosition() {
    if (getBounds().left == 0 && getBounds().width() == getMaxBounds().width()
    && getWindowingMode() == WINDOWING_MODE_FREEFORM) {
        //固定Task的大小
        SurfaceControl.Transaction t = getSyncTransaction();
        Matrix matrix = new Matrix();
        matrix.reset();
        matrix.postScale(0.5f,0.5f);
        matrix.postTranslate(getBounds().width() * 0.5f * 0.5f,getBounds().top);
        if (mFreeFormSurfaceBound == null) {
            mFreeFormSurfaceBound = new Rect();
        }
        mFreeFormSurfaceBound.set(getBounds());
        mFreeFormSurfaceBound.scale(0.5f);
        int deta  = 0 ;
        if (mFreeFormSurfaceBound.top < getBounds().top) {
            deta = getBounds().top -mFreeFormSurfaceBound.top;
        }
        mFreeFormSurfaceBound.offset((int)(getBounds().width() * 0.5f * 0.5f),deta);
        t.setMatrix(getSurfaceControl(),matrix,new float[9]);
        t.apply();
    }
}
//Task.java
void setBoundsByFreeFormSurfaceBound() {
    SurfaceControl.Transaction t = getSyncTransaction();
    float scale = (float)mFreeFormSurfaceBound.width() / (float) getBounds().width();
    t.setScale(getSurfaceControl(),scale,scale);
    t.setPosition(getSurfaceControl(),mFreeFormSurfaceBound.left,mFreeFormSurfaceBound.top
    t.apply();
}
boolean resize(Rect bounds, int resizeMode, boolean preserveWindow) {
    mAtmService.deferWindowLayout();
    try {
        if (inFreeformWindowingMode()) {
            mFreeFormSurfaceBound.set(bounds);
            setBoundsByFreeFormSurfaceBound();
            return true;
        }
    }
```

##### 添加外边框

```java
//DecorCaptionView.java
public boolean mNeedDrawRect = false;
@Override
protected void dispatchDraw(Canvas canvas) {
    super.dispatchDraw(canvas);
    if (mNeedDrawRect) {
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        paint.setAntiAlias(true);
        paint.setStrokeWidth(20);
        paint.setStyle(Paint.Style.STROKE);
        Rect rect = new Rect(0,0,getWidth(),getHeight());
        canvas.drawRect(rect,paint);
    }
}
```

拖动时设置mNeedDrawRect

##### 保持在最顶部

Task的setAlwaysOnTop方法

#### 桌面启动应用全部变freeform模式

设置LaunchWindowingMode(WINDOWING_MODE_FREEFORM)

```java
//ActivityContext.java
default RunnableList startActivitySafely(
            View v, Intent intent, @Nullable ItemInfo item) {
		...
        ActivityOptionsWrapper options = v != null ? getActivityLaunchOptions(v, item)
                : makeDefaultActivityOptions(item != null && item.animationType == DEFAULT_NO_ICON
                        ? SPLASH_SCREEN_STYLE_SOLID_COLOR : -1 /* SPLASH_SCREEN_STYLE_UNDEFINED */);
        UserHandle user = item == null ? null : item.user;
        Bundle optsBundle = options.toBundle();
        ...
            if (isShortcut) {
                // Shortcuts need some special checks due to legacy reasons.
                startShortcutIntentSafely(intent, optsBundle, item);
            } else if (user == null || user.equals(Process.myUserHandle())) {
                //options.options.setLaunchWindowingMode(WINDOWING_MODE_FREEFORM);
              	//不在复用原启动ActivityOptions,复用会显示在左上角盖住状态栏
                ActivityOptions activityOptions = ActivityOptions.makeBasic();
                activityOptions.setLaunchWindowingMode(WINDOWING_MODE_FREEFORM);
                final Rect r = new Rect(0, 0, 1440, (int)(2960 * 0.63f));
                activityOptions.setLaunchBounds(r);
                // Could be launching some bookkeeping activity
                context.startActivity(intent, activityOptions.toBundle());
            } 			
```
