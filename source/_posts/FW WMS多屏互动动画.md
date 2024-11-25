---
title: Android 13 WMS多屏互动
date: 2023-9-20 22:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

#### 功能说明

两个屏幕，主屏幕显示桌面，另一个屏幕显示壁纸。当前打开的应用，双指拖到可以移动到另一个屏幕，应用跟手显示，松开手后移动到另一个屏幕上面。修改代码如下：

```java
DisplayContent.java
DisplayContent(Display display, RootWindowContainer root) {
     	// Tap Listeners are supported for:
        // 1. All physical displays (multi-display).
        // 2. VirtualDisplays on VR, AA (and everything else).
        mTapDetector = new TaskTapPointerEventListener(mWmService, this);
    	//注册全局事件监听
        mDoubleScreenMoveListener = new DoubleScreenMovePointerEventListener(mWmService, this);
        registerPointerEventListener(mTapDetector);
        registerPointerEventListener(mDoubleScreenMoveListener);
}
 public void startAutoMove(int startX,boolean right) {//松手后自动滑动部分实现
        int endX =0;
        if (right) {
            endX = getDisplay().getWidth();
        }
        SurfaceControl.Transaction t =mWmService.mTransactionFactory.get();
        ValueAnimator valueAnimator = ValueAnimator.ofInt(startX,endX);//这里就是方案设计中讲的offsetX ---> width或者offsetX ----> 0
        valueAnimator.addUpdateListener(
                animation -> {
                    android.util.Log.i("DoubleScreen"," animation.getAnimatedValue() = " + animation.getAnimatedValue());
                    int moveX = (int)animation.getAnimatedValue();
                    startMoveCurrentScreenTask(moveX,0);
                });
        valueAnimator.setInterpolator(new AccelerateInterpolator(1.0f));
        valueAnimator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
                if (right) {
                    //对动画播放完成需要进行ActivityRecord状态回置
                    resetState();
                } else {
                    //针对拖动不够太多回到原来
                    mRootWindowContainer.moveRootTaskToDisplay(mCurrentRootTaskId,mDisplayId,true);
                    mCurrentRootTaskId = -1;
                    float[] mTmpFloats = new float[9];
                    Matrix outMatrix = new Matrix();
                    if (realWindowStateBuffer != null) {
                        outMatrix.reset();
                        t.setMatrix(realWindowStateBuffer,outMatrix,mTmpFloats);
                    }
                }
                if (copyTaskSc!= null && copyTaskBuffer!= null) {
                    //动画播放完成则需要把新创建图层进行删除
                    t.remove(copyTaskBuffer);
                    t.remove(copyTaskSc);
                    t.apply();
                    copyTaskSc = null;
                    copyTaskBuffer = null;
                }
            }
        });
        valueAnimator.setDuration(500);
        valueAnimator.start();
    }

    //恢复正常状态，让mLaunchTaskBehind变成false
    void resetState() {
        if (mCurrentRecord != null) {
            mCurrentRecord.mLaunchTaskBehind =  false;
            mRootWindowContainer.ensureActivitiesVisible(null, 0, PRESERVE_WINDOWS);
        }
    }

    //这个是对外触摸的接口
    public void startMoveCurrentScreenTask(int x,int y) {
        if (copyTaskBuffer!= null) {
            //真正调用这个moveCurrentScreenTask相关业务操作
            moveCurrentScreenTask(mWmService.mTransactionFactory.get(),copyTaskBuffer,x,y);
        }
    }

    void moveCurrentScreenTask(SurfaceControl.Transaction t,SurfaceControl surfaceControl,int x,int y) {
        float[] mTmpFloats = new float[9];
        Matrix outMatrix = new Matrix();
        if (realWindowStateBuffer != null) {
            outMatrix.reset();
            //对屏幕2的新task进行坐标平移操作，对屏幕大小一样的则直接就是在个偏移-（width - offsetX） = offsetX - width，屏幕大小不一样则需要进行对应scale操作
            outMatrix.postTranslate(x - mOtherDisplayContent.getDisplayInfo().logicalWidth, y);
            t.show(realWindowStateBuffer);
            t.setMatrix(realWindowStateBuffer,outMatrix,mTmpFloats);//给对应的task图层设置对应的matrix
        }
        outMatrix.reset();
        //这个部分属于屏幕1镜像图层偏移坐标，这里为啥会是这样，不是应该只要x这个偏移就行么？
        //这里其实就和前面说的镜像图层实际挂了task，task再屏幕2进行了坐标改变，当然也会影响屏幕1的镜像图层效果，
        // 所以(getDisplayInfo().logicalWidth - x )是为了消除屏幕2 task的坐标偏移带来的影响，最后屏幕1上的镜像图层偏移量就只是x
        float offsetXMainDisplay =x+ (getDisplayInfo().logicalWidth - x );
        outMatrix.postTranslate(offsetXMainDisplay, y);
        android.util.Log.i("DoubleScreen","moveCurrentScreenTask scaleX =" +" offsetX = "+x
                + " mTmpFloats " + mTmpFloats+ " outMatrix " + outMatrix);
        t.setMatrix(surfaceControl,outMatrix,mTmpFloats);
        t.show(copyTaskBuffer);
        t.apply();
    }
    ActivityRecord mCurrentRecord = null;
    void ensureOtherDisplayActivityVisible(DisplayContent other) {
        //注意这个方法很关键，这里会让activity底下的activity也跟着显示出来，即2个activity同时显示不然拖动task时候底部只能黑屏体验很差
        ActivityRecord otherTopActivity = other.getTopActivity(false,false);
        if (otherTopActivity != null) {
            android.util.Log.i("DoubleScreen","ensureOtherDisplayActivityVisible otherTopActivity = " + otherTopActivity);
            otherTopActivity.mLaunchTaskBehind = true;
            mCurrentRecord = otherTopActivity;
        }
    }
    SurfaceControl copyTaskSc = null;
    SurfaceControl copyTaskBuffer = null;
    SurfaceControl realWindowStateBuffer = null;
    DisplayContent mOtherDisplayContent = null;
    int mCurrentRootTaskId = -1;

    public void doTestMoveTaskToOtherDisplay() {
        DisplayContent otherDisplay = null;
        if (mRootWindowContainer.getChildCount() == 2) {
            otherDisplay = mRootWindowContainer.getChildAt(0) == this
                    ? mRootWindowContainer.getChildAt(1) : mRootWindowContainer.getChildAt(0);
        }
        if (otherDisplay != this && otherDisplay != null) {
            mOtherDisplayContent = otherDisplay;
            try {
                Task rootTask = getTopRootTask();
                if (rootTask == null) {
                    android.util.Log.i("DoubleScreen",
                            "doTestMoveTaskToOtherDisplay rootTask null");
                    return;
                }
                if (rootTask.isActivityTypeHome()) {
                    android.util.Log.i("DoubleScreen",
                            "doTestMoveTaskToOtherDisplay isActivityTypeHome");
                    return;
                }
                int rootTaskId = rootTask.mTaskId;
                //获取task的顶部WindowState
                WindowState windowState = rootTask.getTopActivity(false,false).getTopChild();
                android.util.Log.i("DoubleScreen","doTestMoveTaskToOtherDisplay getTopActivity"
                        + rootTask.getTopActivity(false,false) +" windowState " + windowState);
                if (windowState!= null) {
                    final SurfaceControl.Transaction t = mWmService.mTransactionFactory.get();
                    if (copyTaskSc == null) { //创建一个rootTaskCopy图层主要用来放置镜像Task画面
                        copyTaskSc = makeChildSurface(null)
                                .setName("rootTaskCopy")
                                .setParent(getWindowingLayer())
                                .build();
                    }
                    if (copyTaskBuffer == null) {
                        copyTaskBuffer = SurfaceControl.mirrorSurface(rootTask.getSurfaceControl());
                    }
                    realWindowStateBuffer = rootTask.getSurfaceControl();
                    t.reparent(copyTaskBuffer,copyTaskSc);
                    t.show(copyTaskSc);
                    t.show(copyTaskBuffer);
                    t.apply();
                }
                ensureOtherDisplayActivityVisible(otherDisplay);
                mCurrentRootTaskId = rootTaskId;
                //这里再启动前需要调用一下startMoveCurrentScreenTask，目的是为了把屏幕2的Task坐标移动到屏幕外，
                //不然可能会产生开始拖拉时候，屏幕2会有显示一瞬间的task画面，有个闪烁，这里就早早把偏移设置好
                android.util.Log.i("DoubleScreen","doTestMoveTaskToOtherDisplay startMoveCurrentScreenTask 0");
                startMoveCurrentScreenTask(0,0);
                mRootWindowContainer.moveRootTaskToDisplay(rootTaskId, otherDisplay.mDisplayId, true);
                android.util.Log.i("DoubleScreen","doTestMoveTaskToOtherDisplay end time");
            } catch (Exception e) {
                android.util.Log.i("DoubleScreen", "doTestMoveTaskToOtherDisplay Exception", e);
            }
        }
    }
```

DoubleScreenMovePointerEventListener点击事件处理

```java
public class DoubleScreenMovePointerEventListener implements
        WindowManagerPolicyConstants.PointerEventListener {
    private WindowManagerService mWmService;
    private DisplayContent mDisplayContent;

    public DoubleScreenMovePointerEventListener(WindowManagerService wmService,
            DisplayContent displayContent) {
        this.mWmService = wmService;
        this.mDisplayContent = displayContent;
    }

    private boolean shouldBeginMove = false;
    private int mPoint0FirstX = 0;
    private int mPoint1FirstX = 0;
    private int mPoint0LastX = 0;
    private int mPoint1LastX = 0;
    private int START_GAP = 6;
    private int AUTO_MOVE_GAP = 100;

    @Override
    public void onPointerEvent(MotionEvent motionEvent) {
        android.util.Log.i("DoubleScreen",
                "DoubleScreenMovePointerEventListener onPointerEvent motionEvent = " + motionEvent);
        switch (motionEvent.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
            case MotionEvent.ACTION_POINTER_DOWN:
                if (motionEvent.getPointerCount() > 2) {
                    int detaX = mPoint0LastX - mPoint0FirstX;
                    if (shouldBeginMove && detaX > AUTO_MOVE_GAP) {
                        mDisplayContent.startAutoMove(detaX,detaX > AUTO_MOVE_GAP);
                    }
                    shouldBeginMove = false;
                } else if (motionEvent.getPointerCount() == 2) {
                    if (mPoint0FirstX == 0 && mPoint1FirstX == 0) {
                        mPoint0FirstX = (int) motionEvent.getX(0);
                        mPoint1FirstX = (int) motionEvent.getX(1);
                    }
                }
                break;
            case MotionEvent.ACTION_MOVE:
                if (motionEvent.getPointerCount() == 2) {
                    if (!shouldBeginMove
                            && (motionEvent.getX(0) - mPoint0FirstX > START_GAP
                            && motionEvent.getX(1) - mPoint1FirstX > START_GAP)) {
                        android.util.Log.i("DoubleScreen",
                                "DoubleScreenMovePointerEventListener start DoubleScreenMove1 "
                                        + "mPoint0FirstX=" + mPoint0FirstX + " mPoint1FirstX="
                                        + mPoint1FirstX);
                        shouldBeginMove = true;
                        mDisplayContent.doTestMoveTaskToOtherDisplay();
                    }
                    mPoint0LastX = (int) motionEvent.getX(0);
                    mPoint1LastX = (int) motionEvent.getX(1);
                    if (shouldBeginMove) {
                        int detaX = mPoint0LastX - mPoint0FirstX;
                        android.util.Log.i("DoubleScreen",
                                "DoubleScreenMovePointerEventListener start DoubleScreenMove2 "
                                        + "mPoint0FirstX=" + mPoint0FirstX + " mPoint0LastX="
                                        + mPoint0LastX +" detaX = "+detaX);
                        mDisplayContent.startMoveCurrentScreenTask(detaX, 0);
                    }
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_POINTER_UP:
            case MotionEvent.ACTION_CANCEL:
                if (shouldBeginMove) {
                    int detaX = mPoint0LastX - mPoint0FirstX;
                    mDisplayContent.startAutoMove(detaX,detaX > AUTO_MOVE_GAP);
                }
                shouldBeginMove = false;
                mPoint0FirstX = mPoint1FirstX = 0;
                android.util.Log.i("DoubleScreen",
                        "DoubleScreenMovePointerEventListener ACTION_UP end DoubleScreenMove ");
                break;
        }
    }
}
```

