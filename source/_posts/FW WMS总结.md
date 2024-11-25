---
title: Android 13 WMS总结
date: 2023-9-20 13:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
##### 窗口层级相关

**WindowContainer**

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java
/**
 * Defines common functionality for classes that can hold windows directly or through their
 * children in a hierarchy form.
 * The test class is {@link WindowContainerTests} which must be kept up-to-date and ran anytime
 * changes are made to this class.
 */
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable,
        InsetsControlTarget {
      /**
     * The parent of this window container.
     * For removing or setting new parent {@link #setParent} should be used, because it also
     * performs configuration updates based on new parent's settings.
     */
    private WindowContainer<WindowContainer> mParent = null;

    // List of children for this window container. List is in z-order as the children 	appear on
    // screen with the top-most window container at the tail of the list.
    protected final WindowList<E> mChildren = new WindowList<E>();
   }
```

给可以直接持有窗口的自己或它的孩子定义了一些公共的方法和属性，RootWindowContainer、DisplayContent、DisplayArea、DisplayArea.Tokens、TaskDisplayArea、Task、ActivityRecord、WindowToken、WindowState都是直接或间接的继承该类。

主要的成员变量就是mParent和mChildren，一个代表父节点一个代表子节点，而且子节点的list顺序代表就是z轴的层级显示顺序，list尾巴在比list的头的z轴层级要高。

- RootWindowContainer：根窗口容器，树的根是它。通过它遍历寻找，可以找到窗口树上的窗口。它的孩子是DisplayContent。

- DisplayContent：该类是对应着显示屏幕的，Android是支持多屏幕的，所以可能存在多个DisplayContent对象。
- DisplayArea：该类是对应着显示屏幕下面的，代表一组窗口合集,具有多个子类，如Tokens，TaskDisplayArea等
- TaskDisplayArea：它为DisplayContent的孩子，对应着窗口层次的第2层。第2层作为应用层，看它的定义：int APPLICATION_LAYER = 2，应用层的窗口是处于第2层。TaskDisplayArea的孩子是Task类，其实它的孩子类型也可以是TaskDisplayArea。而Task的孩子则可以是ActivityRecord，也可以是Task。
- Tokens：代表专门包含WindowTokens的容器，它的孩子是WindowToken，而WindowToken的孩子则为WindowState对象。WindowState是对应着一个窗口的。
- ImeContainer：它是输入法窗口的容器，它的孩子是WindowToken类型。WindowToken的孩子为WindowState类型，而WindowState类型则对应着输入法窗口。
- Task：任务，它的孩子可以是Task，也可以是ActivityRecord类型。
- ActivityRecord：是对应着应用进程中的Activity的。ActivityRecord是继承WindowToken的，它的孩子类型为WindowState。
- WindowState：WindowState是对应着一个窗口的。

用dumpsys命令来看看层级结构相关的输出

```
adb shell dumpsys activity containers
```

对应的输出结构树如下

![](https://note.youdao.com/yws/api/personal/file/WEB6cc95a97e35ae4012992ce03150323fc?method=download&shareKey=2c9942b48cce0294715a3d0d2af7df72)

每个显示屏幕的窗口层级分为37层，0-36层。每层可以放置多个窗口，上层窗口覆盖下面的，这个树中每个点一般这样格式：
名字：层级开始数 ：层级结束数

0-36个图层每个图层都有对应节点进行占领，有的一个节点占领一层，如Leaf:36:36 ，有的一个节点可能占领多层如：Leaf:3:12

常用窗口挂载层级：

- Wallpaper处于0-1层
- Activity处于DefaultTaskDisplayArea第2层  后续挂载节点Task->ActivityRecord ->WindowState
- InputMethod处于13-14层
- StatusBar处于15层
- NotificationShade处于17层
- NavigationBar0处于24-25层

##### 结构树构建的源码

**DisplayContent中启动层级树的构建**

```java
    /**
     * Create new {@link DisplayContent} instance, add itself to the root window container and
     * initialize direct children.
     * @param display May not be null.
     * @param root {@link RootWindowContainer}
     */
    DisplayContent(Display display, RootWindowContainer root) {
        super(root.mWindowManager, "DisplayContent", FEATURE_ROOT);
     	...
        final InputChannel inputChannel = mWmService.mInputManager.monitorInput(
                "PointerEventDispatcher" + mDisplayId, mDisplayId);
        mPointerEventDispatcher = new PointerEventDispatcher(inputChannel);
    	//调用surface相关图层配置，因为显示东西需要Surfaceflinger
        final Transaction pendingTransaction = getPendingTransaction();
        configureSurfaces(pendingTransaction);
        pendingTransaction.apply();
        // Sets the display content for the children.
        onDisplayChanged(this);
        ...
    }
    private void configureSurfaces(Transaction transaction) {
     ...
        if (mDisplayAreaPolicy == null) {
            // Setup the policy and build the display area hierarchy.
            // Build the hierarchy only after creating the surface so it is reparented correctly
            //进行DisplayArea构建
            mDisplayAreaPolicy = mWmService.getDisplayAreaPolicyProvider().instantiate(
                    mWmService, this /* content */, this /* root */,
                    mImeWindowsContainer);
        }
    }

```

接下来调用DisplayAreaPolicy中内部类DefaultProvider的instantiate方法

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicy.java
public DisplayAreaPolicy instantiate(WindowManagerService wmService,
                DisplayContent content, RootDisplayArea root,
                DisplayArea.Tokens imeContainer) {
            //创建特殊的TaskDisplayArea，专门来装Activity相关的容器
            final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                    "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
            final List<TaskDisplayArea> tdaList = new ArrayList<>();
            tdaList.add(defaultTaskDisplayArea);

            // Define the features that will be supported under the root of the whole logical
            // display. The policy will build the DisplayArea hierarchy based on this.
    		//构建HierarchyBuilder对象
            final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
            // Set the essential containers (even if the display doesn't support IME).
 			//setImeContainer进行输入法直接容器设置           
 			rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
            if (content.isTrusted()) {//构成层级关键
                // Only trusted display can have system decorations.
                configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
            }
             return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
        }

```

先看下configureTrustedHierarchyBuilder

```java
private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                WindowManagerService wmService, DisplayContent content) {
            // WindowedMagnification should be on the top so that there is only one surface
            // to be magnified.
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
            if (content.isDefaultDisplay) {
                // Only default display can have cutout.
                // See LocalDisplayAdapter.LocalDisplayDevice#getDisplayDeviceInfoLocked.
                rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                        FEATURE_HIDE_DISPLAY_CUTOUT)
                        .all()
                        .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                TYPE_NOTIFICATION_SHADE)
                        .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                                FEATURE_ONE_HANDED)
                                .all()
                                .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL,
                                        TYPE_SECURE_SYSTEM_OVERLAY)
                                .build());
            }
            rootHierarchy
                    .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                            FEATURE_FULLSCREEN_MAGNIFICATION)
                            .all()
                            .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                    TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                    TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                            .build())
                    .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                            FEATURE_IME_PLACEHOLDER)
                            .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                            .build());
        }
    }
```

设置的这些Feature名字,Feature代表的是DisplayArea的一个特征，可以根据Feature来对不同的DisplayArea进行划分。Feature类中成员变量如下：

- mName：这个Feature的名字，如上面的“WindowedMagnification”，“HideDisplayCutout”之类的，后续DisplayArea层级结构建立起来后，每个DisplayArea的名字用的就是当前DisplayArea对应的那个Feature的名字。
- mId：Feature的ID，如上面的FEATURE_WINDOWED_MAGNIFICATION和FEATURE_HIDE_DISPLAY_CUTOUT，虽说是Feature的ID，因为Feature又是DisplayArea的特征
- mWindowLayers：代表了这个DisplayArea可以包含哪些层级对应的窗口

其中一个Feature：

```java
 rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
//typeInclusive代表一个windowType，一般可以通过windowType获取对应的windowLayer，
//获取方法layerFromType，upTo代表逻辑就是把层级范围到typeInclusive
  Builder upTo(int typeInclusive) {
                final int max = layerFromType(typeInclusive, false);
                for (int i = 0; i < max; i++) {
                    mLayers[i] = true;
                }
                set(typeInclusive, true);
                return this;
            }
  /**
             * Set that the feature does not apply to the given window types.
             */
             //排除types
            Builder except(int... types) {
                for (int i = 0; i < types.length; i++) {
                    int type = types[i];
                    set(type, false);
                }
                return this;
            }
            //留下types
             Builder and(int... types) {
                for (int i = 0; i < types.length; i++) {
                    int type = types[i];
                    set(type, true);
                }
                return this;
            }
             private int layerFromType(int type, boolean internalWindows) {
                return mPolicy.getWindowLayerFromTypeLw(type, internalWindows);
            }

```

这里调用了getWindowLayerFromTypeLw来实现窗口类型到层级数的转化：

```java
 default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow,
            boolean roundedCornerOverlay) {
        // Always put the rounded corner layer to the top most.
        if (roundedCornerOverlay && canAddInternalSystemWindow) {
            return getMaxWindowLayer();
        }
        if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
            return APPLICATION_LAYER;
        }

        switch (type) {
            case TYPE_WALLPAPER:
                // wallpaper is at the bottom, though the window manager may move it.
                return  1;
            case TYPE_PRESENTATION:
            case TYPE_PRIVATE_PRESENTATION:
            case TYPE_DOCK_DIVIDER:
            case TYPE_QS_DIALOG:
            case TYPE_PHONE:
                return  3;
            case TYPE_SEARCH_BAR:
                return  4;
            case TYPE_INPUT_CONSUMER:
                return  5;
            case TYPE_SYSTEM_DIALOG:
                return  6;
            case TYPE_TOAST:
                // toasts and the plugged-in battery thing
                return  7;
            case TYPE_PRIORITY_PHONE:
                // SIM errors and unlock.  Not sure if this really should be in a high layer.
                return  8;
            case TYPE_SYSTEM_ALERT:
                // like the ANR / app crashed dialogs
                // Type is deprecated for non-system apps. For system apps, this type should be
                // in a higher layer than TYPE_APPLICATION_OVERLAY.
                return  canAddInternalSystemWindow ? 12 : 9;
            case TYPE_APPLICATION_OVERLAY:
                return  11;
            case TYPE_INPUT_METHOD:
                // on-screen keyboards and other such input method user interfaces go here.
                return  13;
            case TYPE_INPUT_METHOD_DIALOG:
                // on-screen keyboards and other such input method user interfaces go here.
                return  14;
            case TYPE_STATUS_BAR:
                return  15;
            case TYPE_STATUS_BAR_ADDITIONAL:
                return  16;
            case TYPE_NOTIFICATION_SHADE:
                return  17;
            case TYPE_STATUS_BAR_SUB_PANEL:
                return  18;
            case TYPE_KEYGUARD_DIALOG:
                return  19;
            case TYPE_VOICE_INTERACTION_STARTING:
                return  20;
            case TYPE_VOICE_INTERACTION:
                // voice interaction layer should show above the lock screen.
                return  21;
            case TYPE_VOLUME_OVERLAY:
                // the on-screen volume indicator and controller shown when the user
                // changes the device volume
                return  22;
            case TYPE_SYSTEM_OVERLAY:
                // the on-screen volume indicator and controller shown when the user
                // changes the device volume
                return  canAddInternalSystemWindow ? 23 : 10;
            case TYPE_NAVIGATION_BAR:
                // the navigation bar, if available, shows atop most things
                return  24;
            case TYPE_NAVIGATION_BAR_PANEL:
                // some panels (e.g. search) need to show on top of the navigation bar
                return  25;
            case TYPE_SCREENSHOT:
                // screenshot selection layer shouldn't go above system error, but it should cover
                // navigation bars at the very least.
                return  26;
            case TYPE_SYSTEM_ERROR:
                // system-level error dialogs
                return  canAddInternalSystemWindow ? 27 : 9;
            case TYPE_MAGNIFICATION_OVERLAY:
                // used to highlight the magnified portion of a display
                return  28;
            case TYPE_DISPLAY_OVERLAY:
                // used to simulate secondary display devices
                return  29;
            case TYPE_DRAG:
                // the drag layer: input for drag-and-drop is associated with this window,
                // which sits above all other focusable windows
                return  30;
            case TYPE_ACCESSIBILITY_OVERLAY:
                // overlay put by accessibility services to intercept user interaction
                return  31;
            case TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY:
                return 32;
            case TYPE_SECURE_SYSTEM_OVERLAY:
                return  33;
            case TYPE_BOOT_PROGRESS:
                return  34;
            case TYPE_POINTER:
                // the (mouse) pointer layer
                return  35;
            default:
                Slog.e("WindowManager", "Unknown window type: " + type);
                return 3;
        }
    }
```

上面是应用常用的窗口类型，如TYPE_WALLPAPER，TYPE_NAVIGATION_BAR等，其实他们都是有固定的一个层级的。即windowType的值并不是真正层级数目，都是需要通过这个方法进行转化才是真正层级数。

再回到addFeature部分，通过以上的层级获取及相关upTo方法后我们可以得出各个Feature的一个层级情况

|Feature名字|层级情况|
| ---- | ---- |
|WindowedMagnification| 0-31
|HideDisplayCutout| 0-14 16 18-23 26-35
|OneHanded |0-23 26-32 34-35
|FullscreenMagnification| 0-12 15-23 26-27 29-31 33-35
|ImePlaceholder| 13-14

每个Feature对应层级已经清楚了，再接下来就要进入正式的树构建

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicyBuilder.java
   Result build(WindowManagerService wmService) {
        validate();

        // Attach DA group roots to screen hierarchy before adding windows to group hierarchies.
        mRootHierarchyBuilder.build(mDisplayAreaGroupHierarchyBuilders);
        ...
        return new Result(wmService, mRootHierarchyBuilder.mRoot, displayAreaGroupRoots,
                mSelectRootForWindowFunc, mSelectTaskDisplayAreaFunc);
    }
		 /**
         * Builds the {@link DisplayArea} hierarchy below root. And adds the roots of those
         * {@link HierarchyBuilder} as children.
         */
        private void build(@Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
            ...
            PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
            //PendingArea作为root部分
            final PendingArea root = new PendingArea(null, 0, null); 
            //给areaForLayer填满都是默认new PendingArea(null, 0, null); 
            Arrays.fill(areaForLayer, root);
		    //1、创建feature相关的树
            final int size = mFeatures.size();
            for (int i = 0; i < size; i++) {
                // Traverse the features with the order they are defined, so that the early defined
                // feature will be on the top in the hierarchy.
                final Feature feature = mFeatures.get(i);
                PendingArea featureArea = null;
                for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                    if (feature.mWindowLayers[layer]) {
                        // This feature will be applied to this window layer.
                        // We need to find a DisplayArea for it:
                        // We can reuse the existing one if it was created for this feature for the
                        // previous layer AND the last feature that applied to the previous layer is
                        // the same as the feature that applied to the current layer (so they are ok
                        // to share the same parent DisplayArea).
                        //如果featureArea为空，一般每个Feature第一次进入都为null
                        //如果featureArea不为空，featureArea的父节点不一样，即如果兄弟层级featureArea的父节点是						    //同一个那就不需要新创建
                        if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                            // No suitable DisplayArea:
                            // Create a new one under the previous area (as parent) for this layer.
                            //以areaForLayer[layer]为父节点创建一个新的节点
                            featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                            //老容器节点添加新节点
                            areaForLayer[layer].mChildren.add(featureArea);
                        }
                        areaForLayer[layer] = featureArea;
                    } else {
                        // This feature won't be applied to this window layer. If it needs to be
                        // applied to the next layer, we will need to create a new DisplayArea for
                        // that.
                        //如果这一层不支持显示，那么就把featureArea设置为null
                        featureArea = null;
                    }
                }
            }
			//2、创建叶子相关
            // Create Tokens as leaf for every layer.
            PendingArea leafArea = null;
            int leafType = LEAF_TYPE_TOKENS;
            for (int layer = 0; layer < maxWindowLayerCount; layer++) {//遍历36层
                int type = typeOfLayer(policy, layer);
                // Check whether we can reuse the same Tokens with the previous layer. This happens
                // if the previous layer is the same type as the current layer AND there is no
                // feature that applies to only one of them.
                //leafArea空，或者leafArea本身不和这个layer共用一个父节点
                if (leafArea == null || leafArea.mParent != areaForLayer[layer]
                        || type != leafType) {
                    // Create a new Tokens for this layer.
                    leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
                    areaForLayer[layer].mChildren.add(leafArea);
                    leafType = type;
                    //针对属于TYPE_INPUT_METHOD  APPLICATION_LAYER特殊处理
                    if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                        // We use the passed in TaskDisplayAreas for task container type of layer.
                        // Skip creating Tokens even if there is no TDA.
                        ///APPLICATION_LAYER单独处理，就是前面设置过的TaskDisplayArea
                        //设置本身已经设置过的TaskDisplayArea单独处理
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        //设置跳过
                        leafArea.mSkipTokens = true;
                    } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                        // We use the passed in ImeContainer for ime container type of layer.
                        // Skip creating Tokens even if there is no ime container.
                        leafArea.mExisting = mImeContainer;
                        leafArea.mSkipTokens = true;
                    }
                }
                leafArea.mMaxLayer = layer;
            }
            //计算出每个节点最大layer值
            root.computeMaxLayer();

            // We built a tree of PendingAreas above with all the necessary info to represent the
            // hierarchy, now create and attach real DisplayAreas to the root.
            //把PendingArea生成DisplayArea，这里的参数mRoot就是我们的DisplayContent，Root：0：0这个PendingArea
            root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);

            // Notify the root that we have finished attaching all the DisplayAreas. Cache all the
            // feature related collections there for fast access.
            mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);
        }

```

大概步骤如下：

1. 创建PendingArea作为root部分
2. 根据之前的几个Feature的配置进行树图构造
3. 根据窗口层级0~36层，每一层进行遍历，挂载一个新的叶子TOKENS节点，规则和前面Feature一样，如果同一个父亲则不需要新生成，针对TYPE_INPUT_METHOD和APPLICATION_LAYER需要进行特殊处理
4. 把PendingArea生成DisplayArea

##### 添加层级树

1、Task、ActivityRecord的添加
2、普通WindowToken的添加

**Task的添加**

DefaultTaskDisplayArea -> Task ->ActivityRecord

在WindowContainer添加堆栈打印

```java
protected void addChild(E child, Comparator<E> comparator) {
        if(child instanceof Task || child instanceof ActivityRecord || child instanceof WindowState){
            android.util.Log.i("WMSWtf",this + " addChild Comparator c=" + child,new Exception());
        }
}
void addChild(E child, int index) {
        if(child instanceof Task || child instanceof ActivityRecord || child instanceof WindowState){
            android.util.Log.i("WMSWtf",this + " addChild index c=" + child,new Exception());
        }
}
```

打开电话应用堆栈日志

```
DefaultTaskDisplayArea@171686050 addChild index c=Task{56a0fdb #208 type=standard ?? U=0 visible=false visibleRequested=false mode=undefined translucent=true sz=0}
	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:727)
	at com.android.server.wm.TaskDisplayArea.addChildTask(TaskDisplayArea.java:334)
	at com.android.server.wm.TaskDisplayArea.addChild(TaskDisplayArea.java:320)
	at com.android.server.wm.Task$Builder.build(Task.java:6551)
	at com.android.server.wm.TaskDisplayArea.getOrCreateRootTask(TaskDisplayArea.java:1005)
	at com.android.server.wm.TaskDisplayArea.getOrCreateRootTask(TaskDisplayArea.java:1030)
	at com.android.server.wm.RootWindowContainer.getOrCreateRootTask(RootWindowContainer.java:2838)
	at com.android.server.wm.ActivityStarter.getOrCreateRootTask(ActivityStarter.java:3017)
	at com.android.server.wm.ActivityStarter.startActivityInner(ActivityStarter.java:1858)
	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1661)
	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1216)
	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:702)
	at  		com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1240)
	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1203)
	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1178)
	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:893)
	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5183)
	at android.os.Binder.execTransactInternal(Binder.java:1280)
	at android.os.Binder.execTransact(Binder.java:1244)
	
Task{56a0fdb #208 type=standard A=10075:com.android.dialer U=0 visible=false visibleRequested=false mode=fullscreen translucent=true sz=0} addChild index c=ActivityRecord{87c5551 u0
	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:727)
	at com.android.server.wm.TaskFragment.addChild(TaskFragment.java:1835)
	at com.android.server.wm.Task.addChild(Task.java:1429)
	at com.android.server.wm.ActivityStarter.addOrReparentStartingActivity(ActivityStarter.java:2927)
	at com.android.server.wm.ActivityStarter.setNewTask(ActivityStarter.java:2877)
	at com.android.server.wm.ActivityStarter.startActivityInner(ActivityStarter.java:1864)
	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1661)
	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1216)
	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:702)
	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1240)
	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1203)
	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1178)
	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:893)
	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5183)
	at android.os.Binder.execTransactInternal(Binder.java:1280)
	at android.os.Binder.execTransact(Binder.java:1244)
	
ActivityRecord{87c5551 u0 com.android.dialer/.main.impl.MainActivity} t208} addChild Comparator c=Window{736e5ab u0 com.android.dialer/com.android.dialer.main.impl.MainActivity}
	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:694)
	at com.android.server.wm.WindowToken.addWindow(WindowToken.java:302)
	at com.android.server.wm.ActivityRecord.addWindow(ActivityRecord.java:4212)
	at com.android.server.wm.WindowManagerService.addWindow(WindowManagerService.java:1773)
	at com.android.server.wm.Session.addToDisplayAsUser(Session.java:209)
	at android.view.IWindowSession$Stub.onTransact(IWindowSession.java:652)
	at com.android.server.wm.Session.onTransact(Session.java:175)
	at android.os.Binder.execTransactInternal(Binder.java:1285)
	at android.os.Binder.execTransact(Binder.java:1244)
```

**普通WindowToken的添加**

Leaf容器加入StatusBar的WindowToken对应堆栈

```
Leaf:15:15@65133355 addChild Comparator child = WindowToken{9dd676f type=2000 android.os.BinderProxy@ce5e149}
at com.android.server.wm.WindowContainer.addChild
at com.android.server.wm.WindowManagerService.addWindow
at com.android.server.wm.Session.addToDisplayAsUser
```

WindowToken加入对应的StatusBar 的WindowState

```
WindowToken{9dd676f type=2000 android.os.BinderProxy@ce5e149} addChild Comparator child = Window{b94d567 u0 StatusBar}
at com.android.server.wm.WindowContainer.addChild
at com.android.server.wm.WindowToken.addWindow
at com.android.server.wm.WindowManagerService.addWindow
at com.android.server.wm.Session.addToDisplayAsUser
```


##### 添加Window操作

```java
public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, InsetsVisibilities requestedVisibilities,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls) {
       		...
            ActivityRecord activity = null;
            final boolean hasParent = parentWindow != null;
            // Use existing parent window token for child windows since they go in the same token
            // as there parent window so we can apply the same policy on them.
            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
            // If this is a child window, we want to apply the same type checking rules as the
            // parent window type.
            final int rootType = hasParent ? parentWindow.mAttrs.type : type;

            boolean addToastWindowRequiresToken = false;

            final IBinder windowContextToken = attrs.mWindowContextToken;

            if (token == null) {//系统窗口我们在app层面没有设置对应的token故为null
                if (!unprivilegedAppCanCreateTokenWith(parentWindow, callingUid, type,
                        rootType, attrs.token, attrs.packageName)) {
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (hasParent) {
                ...
                } else if (mWindowContextListenerController.hasListener(windowContextToken)) {
                 ...
                } else {//最后进入这里
                    final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                    token = new WindowToken.Builder(this, binder, type)//构造对应的WindowToken容器
                            .setDisplayContent(displayContent)
                            .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                            .setRoundedCornerOverlay(isRoundedCornerOverlay)
                            .build();
                }
            } 
             ...
             //基于token构造对应WindowState
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
          
            //校验一些策略
            final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
            displayPolicy.adjustWindowParamsLw(win, win.mAttrs);
            attrs.flags = sanitizeFlagSlippery(attrs.flags, win.getName(), callingUid, callingPid);
            attrs.inputFeatures = sanitizeSpyWindow(attrs.inputFeatures, win.getName(), callingUid,
                    callingPid);
            win.setRequestedVisibilities(requestedVisibilities);
            //检验是否有权限加入
            res = displayPolicy.validateAddingWindowLw(attrs, callingPid, callingUid);
            if (res != ADD_OKAY) {
                return res;
            }
           //开启与InputDispatcher通信的Socketpair
            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }

            ...
            win.attach();
            //装入容器，后面可以根据binder直接获取winstate
            mWindowMap.put(client.asBinder(), win);
            win.initAppOpsState();
   			...

            win.mToken.addWindow(win);
            displayPolicy.addWindowLw(win, attrs);
            displayPolicy.setDropInputModePolicy(win, win.mAttrs);
            ...     
        }
        Binder.restoreCallingIdentity(origId);
        return res;
    }
```

总结addWindow主要干了以下几件事：

1. 创建WindowToken并挂载到对应的节点层级
2. WindowState初始化和相关变量设置校验
3. 开启与InputDispatcher通信的Socketpair
4. WindowState挂载到WindowToken

##### relayout操作

```java
public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            ClientWindowFrames outFrames, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Bundle outSyncIdBundle) {
   			//根据client找到对应的WindowState 
            final WindowState win = windowForClientLocked(session, client, false);
            ...
            // Create surfaceControl before surface placement otherwise layout will be skipped
            // (because WS.isGoneForLayout() is true when there is no surface.
            if (shouldRelayout) {
                try {
                    //创建对应的window需要的SurfaceControl，传递回应用，应用用他进行绘制
                    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
                } catch (Exception e) {
                  ...
                }
            }
			//调用最核心的performSurfacePlacement进行相关的layout操作
            // We may be deferring layout passes at the moment, but since the client is interested
            // in the new out values right now we need to force a layout.
            mWindowPlacerLocked.performSurfacePlacement(true /* force */);
			...
                
            if (!win.isGoneForLayout()) {
                win.mResizedWhileGone = false;
            }
 			//把相关config数据填回去
            win.fillClientWindowFramesAndConfiguration(outFrames, mergedConfiguration,
                    false /* useLatestConfig */, shouldRelayout);
            //把相关inset数据设置回去
            outInsetsState.set(win.getCompatInsetsState(), win.isClientLocal());
            ...
            getInsetsSourceControls(win, outActiveControls);
        }

        Binder.restoreCallingIdentity(origId);
        return result;
    }

```

主要就有2个最关键步骤：
1、创建对应Window的SurfaceControl
2、计算出对应的window区域等，把inset和config传递回去

##### 窗口动画

Animator 对SurfaceA进行动画控制Matrix(放大缩小位移)alpha圆角等

- LocalAnimation：systemserver自己在WindowAnimator类进行动画播放控制
- RemoteAnimation：systemserver负责提供对应的SurfaceControl给远端进程让远端进行动画播放控制

窗口动画执行执行过程中会在窗口层级树中窗口节点和父节点中间插入一个leash节点对窗口进行控制，播放结束移除节点

##### Activity添加显示过程

1. Activity#attach()方法之内PhoneWindow被创建，并同时创建一WindowManagerImpl负责维护PhoneWindow内的内容。  

2. 在Activity#onCreate()中调用setContentView()方法，这个方法内部创建一个DecorView实例作为PhoneWindow的实体内容。		         

3. WindowManagerImpl决定管理DecorView，并创建一个ViewRootImpl实例,将ViewRootImpl与View树进行关联，这样ViewRootImpl就可以指挥View树的具体工作。
4. Activity resume时调用WindowManagerImpl.addView然后一步步调用WMS中的Window添加。

DecorView是FrameLayout的子类，它可以被认为是Android视图树的根节点视图。

在Activity的生命周期onCreate方法里面，是会设置好布局内容通过setContentView(布局id)的方式，这里是通过xml解析器转化为一个View，这个View会被添加到ContentView中去，成为唯一的子View。

WindowManagerGlobal是一个单例类，一个进程只有一个实例。它管理者所有Window的ViewRootImpl、DecorView、LayoutParams。

ViewRootImpl是View树的树根，但它却又不是View，实现了View与WindowManager之间的通信协议，在WindowManagerGloble中的addView中被建立，是顶层DecorView的ViewParent

- View树的树根并管理View树

- 触发View的测量、布局和绘制

- 输入响应的中转站

- 负责与WMS进行进程间通信。

WMS是窗口的管理者，它负责窗口的启动、添加和删除，窗口动画。另外窗口的大小和层级也是由WMS进行管理的。WMS中对应的窗口在SurfaceFlinger中都有对应的layer对应，后续有机会看SurfaceFlinger代码再具体分析，还有窗口相关的事件分发处理。
