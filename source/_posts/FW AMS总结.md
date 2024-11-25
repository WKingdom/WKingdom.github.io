---
title: AMS及Activity启动总结
date: 2023-9-19 23:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

ActivityManagerService是Android系统中一个特别重要的系统服务，由SytemServer启动，运行在SytemServer进程。也是上层APP打交道最多的系统服务之一。ActivityManagerService（以下简称AMS） 主要负责**四大组件的启动、切换、调度以及应用进程的管理和调度工作**。所有的APP应用都需要与AMS打交道。

Activity Manager的组成主要分为以下几个部分：

1. 服务代理：由ActivityManagerProxy实现，用于与Server端提供的系统服务进行进程间通信
2. 服务中枢：ActivityManagerNative继承自Binder并实现IActivityManager，它提供了服务接口和Binder接口的相互转化功能，并在内部存储服务代理对像，并提供了getDefault方法返回服务代理
3. Client：由ActivityManager封装一部分服务接口供Client调用。ActivityManager内部通过调用ActivityManagerNative的getDefault方法，可以得到一个ActivityManagerProxy对像的引用，进而通过该代理对像调用远程服务的方法
4. Server:由ActivityManagerService实现，提供Server端的系统服务

#### AMS的启动过程

AMS是在SystemServer中被添加的， 所以先到SystemServer中查看初始化

```java
public static void main(String[] args) {
        new SystemServer().run();
    }
private void run() {
    createSystemContext();
    // Create the system service manager.
    mSystemServiceManager = new 
	SystemServiceManager(mSystemContext);
    mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
}
```

在SystemServer中，在startBootstrapServices()中去启动了AMS

```java
private void startBootstrapServices() {
    Installer installer = mSystemServiceManager.startService(Installer.class);
    mSystemServiceManager.startService(PowerStatsService.class);
    // Activity manager runs the show.
     t.traceBegin("StartActivityManager");
    //启动ATMS和AMS
     ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
     mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
     mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
     mActivityManagerService.setInstaller(installer);
     mWindowManagerGlobalLock = atm.getGlobalLock();
        t.traceEnd();
     // Set up the Application instance for the system process and get started.
     t.traceBegin("SetSystemProcess");
     mActivityManagerService.setSystemProcess();
}
```

AMS是通过SystemServiceManager.startService去启动的，参数是ActivityManagerService.Lifecycle.class， 首先看看startService方法

```java
frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
 public <T extends SystemService> T startService(Class<T> serviceClass) {
    Constructor<T> constructor = serviceClass.getConstructor(Context.class);
    service = constructor.newInstance(mContext);
    startService(service);
    return service;
}
    public void startService(@NonNull final SystemService service) {
        // Check if already started
        String className = service.getClass().getName();
        mServiceClassnames.add(className);
        // Register it.
        mServices.add(service);
        service.onStart();
    }
```

startService方法很简单，是通过传进来的class然后反射创建对应的service服务。所以此处创建的是Lifecycle的实例， 然后通过startService启动了AMS服务，那再去看看ActivityManagerService.Lifecycle这个类的构造方法

```java
 public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        } 
        @Override
        public void onStart() {
            mService.start();
        }
    
        public ActivityManagerService getService() {
            return mService;
        }
    }
```

再来看下AMS初始化做了什么

```java
   public ActivityManagerService(Context systemContext) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
        mInjector = new Injector();
        mContext = systemContext;//赋值mContext
        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread();//获取当前的ActivityThread
        mUiContext = mSystemThread.getSystemUiContext();//赋值mUiContext
 		//创建Handler线程，用来处理handler消息
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        mUiHandler = mInjector.getUiHandler(this);//处理ui相关msg的Handler

        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
        mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());
		//管理AMS的一些常量，厂商定制系统就可能修改此处
        mConstants = new ActivityManagerConstants(this, mHandler);

        /* static; one-time init here */
        if (sKillHandler == null) {
            sKillThread = new ServiceThread(TAG + ":kill",
                    THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
            sKillThread.start();
            sKillHandler = new KillHandler(sKillThread.getLooper());
        }
		//初始化管理前台、后台广播的队列， 系统会优先遍历发送前台广播
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
		//初始化管理Service的 ActiveServices对象
        mServices = new ActiveServices(this);
        mProviderMap = new ProviderMap(this);//初始化Provider的管理者
        mAppErrors = new AppErrors(mUiContext, this);//初始化APP错误日志的打印器
 		//创建电池统计服务， 并输出到指定目录
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();
        mAppWarnings = new AppWarnings(this, mUiContext, mHandler, mUiHandler, systemDir);

        // TODO: Move creation of battery stats service outside of activity manager service.
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, mHandler);
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
         //创建进程统计分析服务，追踪统计哪些进程有滥用或不良行为       : 
		mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);
		//加载Uri的授权文件
        mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"), "uri-grants");
		//负责管理多用户
        mUserController = new UserController(this);
		//vr功能的控制器
        mVrController = new VrController(this);
    	//初始化OpenGL版本号
        GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
            ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
        mTempConfig.setToDefaults();
        mTempConfig.setLocales(LocaleList.getDefault());
        mConfigurationSeq = mTempConfig.seq = 1;
        //管理ActivityStack的重要类，这里面记录着activity状态信息，是AMS中的核心类
        mStackSupervisor = createStackSupervisor();
        mStackSupervisor.onConfigurationChanged(mTempConfig);
        //根据当前可见的Activity类型，控制Keyguard遮挡，关闭和转换。 Keyguard就是我们的锁屏相关页面    
        mKeyguardController = mStackSupervisor.getKeyguardController();
        //管理APK的兼容性配置解析/data/system/packages-compat.xml文件，该文件用于存储那些需要考虑屏幕尺寸的APK信息，
        mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
        //Intent防火墙，Google定义了一组规则，来过滤intent，如果触发了，则intent会被系统丢弃，且不会告知发送者
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
        mTaskChangeNotificationController =
                           new TaskChangeNotificationController(this, mStackSupervisor, mHandler);
        //这是activity启动的处理类，这里管理者activity启动中用到的intent信息和flag标识，也和stack和task有重要的联系     
        mActivityStartController = new ActivityStartController(this);
        mRecentTasks = createRecentTasks();
        mStackSupervisor.setRecentTasks(mRecentTasks);
        mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mHandler);
        mLifecycleManager = new ClientLifecycleManager();
    	//启动一个线程专门跟进cpu当前状态信息，AMS对当前cpu状态了如指掌，可以更加高效的安排其他工作
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                synchronized (mProcessCpuTracker) {
                    mProcessCpuInitLatch.countDown();
                    mProcessCpuTracker.init();
                }
                while (true) {
                    updateCpuStatsNow();
               }
        };

        mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);
		//看门狗，监听进程。这个类每分钟调用一次监视器。 如果进程没有任何返回就杀掉
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
        //更新进程优先级，根据当前内存状态和优先级判断是否需要kill掉进程
        updateOomAdjLocked();
     }
```

```java
private void start() {
        removeAllProcessGroups();
        mProcessCpuThread.start();
        mBatteryStatsService.publish();
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
        // Wait for the synchronized block started in mProcessCpuThread,
        // so that any other acccess to mProcessCpuTracker from main thread
        // will be blocked during mProcessCpuTracker initialization.
         //等待mProcessCpuThread完成初始化后， 释放锁，初始化期间禁止访问
        try {
            mProcessCpuInitLatch.await();
        } catch (InterruptedException e) {
            Slog.wtf(TAG, "Interrupted wait during start", e);
            Thread.currentThread().interrupt();
        }
    }
```

再看下setSystemProcess 干了什么

```java
 public void setSystemProcess() {
        try {
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH);
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this)/* allowIsolated= */ false, 
								DUMP_FLAG_PRIORITY_CRITICAL);
     		}
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));
            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
            synchronized (this) {
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;
                app.pid = MY_PID;
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
        // Start watching app ops after we and the package manager are up and running.
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (mAppOpsService.checkOperation(op, uid, packageName)
                                   != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });
    }
```

注册服务。首先将ActivityManagerService注册到ServiceManager中，其次将几个与系统性能调试相关的服务注册到ServiceManager。

查询并处理ApplicationInfo。首先调用PackageManagerService的接口，查询包名为android的应用程序的ApplicationInfo信息，对应于framework-res.apk。然后以该信息为参数调用ActivityThread上的installSystemApplicationInfo方法。

创建并处理ProcessRecord。调用ActivityManagerService上的newProcessRecordLocked，创建一个 ProcessRecord类型的对象，并保存该对象的信息。

#### AMS是什么

1. 从java角度来看，AMS就是一个java对象，实现了Ibinder接口，所以它是一个用于进程之间通信的 接口，这个对象初始化是在systemServer.java 的run()方法里面。
2. AMS是一个服务，ActivityManagerService从名字就可以看出，它是一个服务，用来管理Activity，而且是一个系统服务
3. AMS是一个Binder，AMS实现了Ibinder接口，所以它是一个Binder，这意味着他不但可以用于进程间通信，还是一个线程，因为一个Binder就是一个线程。

#### AMS相关重要类介绍

##### ProcessRecord

**第一类数据：描述身份的数据**

1. ApplicationInfo info：AndroidManifest.xml中定义的Application信息 
2. boolean isolated：是不是isolated进程 
3. int uid：进程uid 
4. int userId：这个是android做的多用户系统id，就像windows可以登录很多用户一样，android 也希望可以实现类似的多用户 
5. String processName：进程名字，默认情况下是包名 
6. UidRecord uidRecord：记录已经使用的uid 
7. IApplicationThread thread：这个很重要，它是ApplicationThread的客户端，AMS就是通过这个对象给apk进程发送异步消息的（管理四大组件的消息），所以只有这个对象不为空的情况下， 才代表apk进程可是使用了 
8. int pid：进程的pid 
9. String procStatFile：proc目录下每一个进程都有一个以pid命名的目录文件，这个目录下记载着 进程的详细信息，这个目录及目录下的文件是内核创建的， proc是内核文件系统，proc就是 process的缩写，涉及的目的就是导出进程内核信息 
10. int[] gids：gid组 
11. CompatibilityInfo compat : 兼容性信息 
12. String requiredAbi : abi信息 
13. String instructionSet : 指令集信息

**第二类数据：描述进程中组件的数据**

1. pkgList：进程中运行的包 
2. ArraySet pkgDeps：进程运行依赖的包 
3. ArrayList activities：进程启动的所有的activity组件记录表 
4. ArraySet services：进程启动的所有的service组件记录表 
5. ArraySet executingServices：正在运行（executing）是怎么定义的？首先需要明确的是系统是怎么控制组件的？发送消息给apk进程，apk进程处理消息，上报消息完成，这被定义为一个完整的执行过程，因此正在执行（executing）被定义为发送消息到上报完成这段时间 
6. ArraySet connections：绑定service的客户端记录表 
7. ArraySet receivers：广播接收器的记录表 
8. ContentProviderRecord pubProviders：pub是publish（发布）的意思，ContentProvider需要 安装然后把自己发布到系统（AMS）中后，才能使用，安装指的是apk进程加载ContentProvider 子类、初始化、创建数据库等过程，发布是将ContentProvider的binder客户端注册到AMS中 
9. ArrayList conProviders：使用ContentProvider的客户端记录表 
10. BroadcastRecord curReceiver：当前进程正在执行的广播

**第三类数据：描述进程状态的数据**

1. int maxAdj：进程的adj上限（adjustment） 
2. int curRawAdj：当前正在计算的adj，这个值有可能大于maxAdj 
3. int setRawAdj：上次计算的curRawAdj设置到lowmemorykiller系统后的adj 
4. int curAdj：当前正在计算的adj，这是curRawAdj被maxAdj削平的值 
5. int setAdj：上次计算的curAdj设置到lowmemorykiller系统后的adj 
6. int verifiedAdj：setAdj校验后的值 
7. int curSchedGroup：正在计算的调度组 
8. int setSchedGroup：保存上次计算的调度组 
9. int curProcState：正在计算的进程状态 
10. int repProcState：发送给apk进程的状态 
11. int setProcState：保存上次计算的进程状态 
12. int pssProcState：pss进程状态 
13. ProcessState baseProcessTracker：进程状态监测器 
14. int adjSeq：计算adj的序列数 
15. int lruSeq：lru序列数 
16. IBinder forcingToForeground:强制将进程的状态设置为前台运行的IBinder，IBinder代表的是 组件的ID，这个是整个android系统唯一

**第四类数据：和pss相关的数据**

1. long initialIdlePss：初始化pss 
2. long lastPss：上次pss 
3. long lastSwapPss：上次SwapPss数据
4. long lastCachedPss：上次CachedPss数据 
5. long lastCachedSwapPss：上次CachedSwapPss数据

```
VSS- Virtual Set Size 虚拟耗用内存（包含共享库占用的内存）
RSS- Resident Set Size 实际使用物理内存（包含共享库占用的内存）
PSS- Proportional Set Size 实际使用的物理内存（比例分配共享库占用的内存）
USS- Unique Set Size 进程独自占用的物理内存（不包含共享库占用的内存）
一般来说内存占用大小有如下规律：VSS >= RSS >= PSS >= USS
```

**第五类数据：和时间相关的数据**

1. long lastActivityTime：上次使用时间
2. long lastPssTime：上次计算pss的时间 
3. long nextPssTime：下次计算pss的时间 
4. long lastStateTime：上次设置进程状态的时间
5. long lastWakeTime：持有wakelock的时长
6. long lastCpuTime：上次计算占用cpu的时长 
7. long curCpuTime：当前最新占用cpu的时长 
8. long lastRequestedGc：上次发送gc命令给apk进程的时间 
9. long lastLowMemory：上次发送低内存消息给apk进程的时间 
10. long lastProviderTime：上次进程中ContentProvider被使用的时间 
11. long interactionEventTime：上次发送交互时间时间 
12. long fgInteractionTime：变成前台的时间

**第六类数据：crash和anr相关的数据**

1. IBinder.DeathRecipient deathRecipient：apk进程退出运行的话，会触发这个对象的 binderDied()方法，来回收系统资源 
2. boolean crashing：进程已经crash
3. Dialog crashDialog：crash对话框 
4. boolean forceCrashReport：强制crash对话框显示 
5. boolean notResponding：是否处于anr状态
6. Dialog anrDialog：anr显示对话框 
7. Runnable crashHandler：crash回调
8. ActivityManager.ProcessErrorStateInfo crashingReport:crash报告的进程状态
9. ActivityManager.ProcessErrorStateInfo notRespondingReport:anr报告的进程状态
10. String waitingToKill:后台进程被kill原因 
11. ComponentName errorReportReceiver:接收error信息的组件

**第七类数据：和instrumentation相关的数据**

instrumentation 也可以说是apk的一个组件，如果提供的话，系统会默认使用 Instrumentation.java类，按照我们一般的理解，UI 线程控制activity的生命周期，是直接调用Activity 类的方法，实际是，UI线程调用的是instrumentation的方法，由它在调用Activity涉及生命周期的方法，所有如果我们覆写了instrumentation的这些方法，就可以了解所有的Activity的生命周期了。

1. ComponentName instrumentationClass：AndroidManifest.xml中定义的instrumentation信 息 
2. ApplicationInfo instrumentationInfo：instrumentation应用信息 
3. String instrumentationProfileFile：instrumentation配置文件 
4. IInstrumentationWatcher instrumentationWatcher：instrumentation监测器 
5. IUiAutomationConnection instrumentationUiAutomationConnection：UiAutomation连接器
6. ComponentName instrumentationResultClass：返回结果组件

**第八类数据：电源信息和调试信息**

1. BatteryStatsImpl mBatteryStats:电量信息 
2. BatteryStatsImpl.Uid.Proc curProcBatteryStats：当前进程电量信息 
3. boolean debugging:处于调试中
4. boolean waitedForDebugger:等待调试 
5. Dialog waitDialog:等待对话框 
6. String adjType:adj类型（或者说标示） 
7. int adjTypeCode:adj类型码（也是一种标示）
8. Object adjSource:改变adj的组件记录表 
9. int adjSourceProcState:影响adj的进程状态 
10. Object adjTarget：改变adj的组件 
11. String shortStringName：进程记录表的字符串显示 
12. String stringName：进程记录表的字符串显示

**第九类数据：一堆boolean类型的值**

**进程声明周期相关的**

a.boolean starting:进程正在启动 

b.boolean removed:进程系统资源已经清理 

c.boolean killedByAm:进程被AMS主动kill掉 

d.boolean killed:进程被kill掉了 

e:boolean persistent:常驻内存进程

**组件状态影响进程行为的**

a.boolean empty:空进程，不含有任何组件的进程 

b.boolean cached:缓存进程 

c.boolean bad:60s内连续crash两次的进程被定义为bad进程 

d.boolean hasClientActivities:进程有Activity绑定其他Service 

e.boolean hasStartedServices:进程中包含启动了的Service 

f.boolean foregroundServices:进程中包含前台运行的Service 

g.boolean foregroundActivities:进程中包含前台运行的Activity 

h.boolean repForegroundActivities: 

i.boolean systemNoUi:系统进程，没有显示UI 

j.boolean hasShownUi:重进程启动开始，是否已经显示UI 

k.boolean pendingUiClean: 

l.boolean hasAboveClient:进程中有组件使用BIND_ABOVE_CLIENT标志绑定其他Service 

m.boolean treatLikeActivity:进程中有组件使用BIND_TREAT_LIKE_ACTIVITY标志绑定其他 Service 

n.boolean execServicesFg:前台执行Service 

o.boolean setIsForeground:设置运行前台UI

还有一些其他的。

##### ProcessRecord容器

**永久性容器**

1. mProcessNames：根据进程名字检索进程记录表 
2. mPidsSelfLocked：根据进程pid检索进程记录表 
3. mLruProcesses：lru进程记录表容器，这个容器使用的是最近最少使用算法对进程记录表进行排 序，越是处于上层的越是最近使用的，对于系统来说就是最重要的，在内存吃紧回收进程时，越不 容易被回收，实现起来也很简单

**临时性容器**

1. mPersistentStartingProcesses：常驻内存进程启动时容器 
2. mProcessesOnHold：进程启动挂起容器 
3. mProcessesToGc：将要执行gc回收的进程容器 
4. mPendingPssProcesses：将要计算Pss数据的进程容器

mRemovedProcesses：已经移除的进程

##### 内部四大组件记录表的容器

组件运行才是进程存在的意义，由于android系统进程间的无缝结合，所以系统需要控制到组件级别， 所有的组件信息都需要映射到系统，

一个ActivityRecord记录对应一个Activity的信息

一个 ServiceRecord记录对应一个Service的信息

一个ConnectionRecord记录对应一个bind service的客户端信息

一个ReceiverList对应处理同一事件的一组广播

一个ContentProviderRecord记录对应一个 ContentProvider信息

一个ContentProviderConnection对应一个进程中的所有ContentProvider客户端

**activity记录**

  activities：ActivityRecord的容器，进程启动的所有的activity组件记录表

**service记录**

services：ServiceRecord的容器，进程启动的所有的service组件记录表

executingServices：正在运行（executing）的ServiceRecord是怎么定义的？首先需要明确的是 系统是怎么控制组件的？发送消息给apk进程，apk进程处理消息，上报消息完成，这被定义为一 个完整的执行过程，因此正在执行（executing）被定义为发送消息到上报完成这段时间

connections：ConnectionRecord容器，绑定service的客户端记录表

**广播接收器记录**

receivers：ReceiverList容器，广播接收器的记录表

**ContentProvider记录**

pubProviders：名字到ContentProviderRecord的映射容器，pub是publish（发布）的意思， ContentProvider需要安装然后把自己发布到系统（AMS）中后，才能使用，安装指的是apk进程 加载ContentProvider子类、初始化、创建数据库等过程，发布是将ContentProvider的binder客户端注册到AMS中

conProviders：ContentProviderConnection容器，使用ContentProvider的客户端记录表

#### 与Activity管理有关的数据结构

##### **ActivityRecord**

ActivityRecord，源码中的注释介绍：An entry in the history stack, representing an activity. 翻译：历史栈中的一个条目，代表一个activity。

ActivityRecord中存在着大量的成员变量，包含了一个Activity的所有信息。 ActivityRecord中的成员变量task表示其所在的TaskRecord，由此ActivityRecord与 TaskRecord建立了联系。

##### **TaskRecord**

TaskRecord，内部维护一个ArrayList用来保存ActivityRecord，TaskRecord中的mStack表示其所在的ActivityStack。 startActivity()时也会创建一个TaskRecord

##### **ActivityStack**

ActivityStack,内部维护了一个ArrayList，用来管理TaskRecord，ActivityStack中还持有ActivityStackSupervisor对象，这个是用来管理ActivityStacks的。 ActivityStack是由ActivityStackSupervisor来创建的，实际ActivityStackSupervisor就是用来管理 ActivityStack的

##### **ActivityStackSupervisor**

ActivityStackSupervisor内部有两个不同的ActivityStack对象：mHomeStack、mFocusedStack，用来管理不同的任务。 ActivityStackSupervisor内部包含了创建ActivityStack对象的方法。 AMS初始化时会创建一个ActivityStackSupervisor对象。

#### Activity启动流程相关

整体流程

![](https://note.youdao.com/yws/api/personal/file/WEB62d1c2c390d165de0619802e5cb17dca?method=download&shareKey=ec2c8cca3887831b1da957242a1488e2)

以Launcher打开应用为例，整体流程如下：

1、Launcher通过ActivityTaskManger向ATMS请求打开应用

2、ATMS内部判断进程是否存在，不存在通知Zygote fork一个新的进程

3、Zygote fork新的进程启动后，把ApplicationThread attach到AMS

4、AMS绑定后执行Activity的启动，ATMS执行启动流程，向APP发生启动事件

5、APP处理启动事件,执行Activity的生命周期。

打开应用，进程存在的情况下，Activity启动流程都是一样的，不同的是目标进程是否存在的情况下，下面先看下进程不存在的情况下，进程创建的流程，就是上面总体流程图中的2，3，4三步。

**AMS通知Zygote fork新进程的过程**

![img](https://note.youdao.com/yws/api/personal/file/WEBaa1c4fe17c224e325639f885fef52660?method=download&shareKey=e60495f3493124e547d4073e9bd30511)

**Application进程启动后，attach过程**

![](https://note.youdao.com/yws/api/personal/file/WEBc82c8db12fd026588095950d7958cabc?method=download&shareKey=5951bef4ed57d94231108c59251e2a4e)

上面的两个时序图是Zygote fork新进程启动后，然后把app的IBinder绑定到了AMS的过程。ApplicationThread实现了IApplicationThread.Stub，是y一个Ibinder类，就是这个类在负责接收AMS的消息。

进程已经存在后面再梳理下Activity是如何启动的。大流程可以分为下面五步：

1、启动Activity端，准备启动参数阶段，Intent处理，前置条件判断

2、AMS端,处理ActivityStack阶段，产生ClientTransaction

3、跨进程传递生命周期触发器ClientTransaction阶段，传递给目标Activity端

4、目标Activity端生命周期触发器ClientTransaction触发执行阶段

5、启动Activity端的Activity stop阶段

**1、准备启动参数阶段**

![](https://note.youdao.com/yws/api/personal/file/WEB1f25ca857b41066914721815deba3909?method=download&shareKey=399142ee47825577e78e20757ed74070)

**2、处理ActivityStack阶段**

![](https://note.youdao.com/yws/api/personal/file/WEB52d9a565b86fd421dc0d62973d8f7638?method=download&shareKey=d09ce60f49af3f816b28316dd7baab9c)

**3、跨进程传递生命周期触发器**

![](https://note.youdao.com/yws/api/personal/file/WEB05c68ed2c089fe91b6d1f1d4b40203d5?method=download&shareKey=8ac5d63585e331d8863a8d6723410194)

**4、ClientTransaction触发执行阶段**

![](https://note.youdao.com/yws/api/personal/file/WEB062f6b846da88d8d20fcbb9dbc40c0ed?method=download&shareKey=6883431c750163c5741e584c9e0e0321)

**5、Activity stop阶段**

![](https://note.youdao.com/yws/api/personal/file/WEBfb6b0c62016f88e889ed0cbf33fe1a26?method=download&shareKey=388d461e03c6ee69a1d4d71d6c8e2ece)

方法很多，具体细节可以按照上面的时序图结合代码去过一遍。

adb shell dumpsys activity activities | sed -En -e '/Stack #/p' -e '/Running activities/,/Run #0/p'
