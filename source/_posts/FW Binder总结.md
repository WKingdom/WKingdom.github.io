---
title: Binder总结
date: 2023-9-17 23:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

跨进程通信是需要内核空间做支持的。传统的 IPC 机制如管道、Socket 都是内核的一部分，因此通过内核支持来实现进程间通信自然是没问题的。但是 Binder 并不是 Linux 系统内核的一部分，那怎么办呢？这就得益于 Linux的动态内核可加载模块（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译， 但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。

 在 Android 系统中，这个运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 Binder驱 动（Binder Dirver）。

 那么在 Android 系统中用户进程之间是如何通过这个内核模块（Binder 驱动）来实现通信的呢？难道是和前面说的 传统 IPC 机制一样，先将数据从发送方进程拷贝到内核缓存区，然后再将数据从内核缓存区拷贝到接收方进程，通过 两次拷贝来实现吗？显然不是。

 这就涉及到Linux 下的另一个概念：**内存映射**。 

Binder IPC 机制中涉及到的内存映射通过 **mmap()** 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。

内存映射能减少数据拷贝次数，实现用户空间和内核空间的高效互动。两个空间各自的修改能直接反映在映射的内存区域，从而被对方空间及时感知。也正因为如此，内存映射能够提供对进程间通信的支持。

 **Binder IPC 正是基于内存映射（mmap）来实现的，但是 mmap() 通常是用在有物理介质的文件系统上的。** 

比如进程中的用户区域是不能直接和物理设备打交道的，如果想要把磁盘上的数据读取到进程的用户区域，需要两次 拷贝（磁盘-->内核空间-->用户空间）；通常在这种场景下 mmap() 就能发挥作用，通过在物理介质和用户空间之间 建立映射，减少数据的拷贝次数，用内存读写取代I/O读写，提高文件读取效率。 

而 Binder 并不存在物理介质，因此 Binder 驱动使用 mmap() 并不是为了在物理介质和用户空间之间建立映射，而是在内核空间创建数据接收的缓存空间。

 一次完整的 Binder IPC 通信过程通常是这样：

1. 首先 Binder 驱动在内核空间创建一个数据接收缓存区
2. 接着在内核空间开辟一块内核缓存区，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系；发送方进程通过系统调用 
3. copyfromuser() 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信

![](https://note.youdao.com/yws/api/personal/file/WEB05a2be2615b8aafb64509e6b80d27c9c?method=download&shareKey=ae7691508872565960006bd052486f4f)

**性能**
首先说说性能上的优势。Socket 作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。Binder 只需要一次数据拷贝，性能上仅次于共享内存。
**稳定性**
再说说稳定性，Binder 基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责明确又相互独立，自然稳定性更好。共享内存虽然无需拷贝，但是控制负责，难以使用。从稳定性的角度讲，Binder 机制是优于内存共享的。
**安全性** 

另一方面就是安全性。Android 作为一个开放性的平台，市场上有各类海量的应用供用户选择安装，因此安全性对于Android 平台而言极其重要。作为用户当然不希望我们下载的 APP 偷偷读取我的通信录，上传我的隐私数据，后台偷跑流量、消耗手机电量。传统的 IPC 没有任何安全措施，完全依赖上层协议来确保。首先传统的 IPC 接收方无法获得对方可靠的进程用户ID/进程ID（UID/PID），从而无法鉴别对方身份。Android 为每个安装好的 APP 分配了自己的 UID，故而进程的 UID 是鉴别进程身份的重要标志。传统的 IPC 只能由用户在数据包中填入 UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。其次传统的 IPC 访问接入点是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。同时Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。
基于上述原因，Android 需要建立一套新的 IPC 机制来满足系统对稳定性、传输性能和安全性方面的要求，这就是Binder。

#### Binder何时初始化 

Binder初始化一般是指binder驱动的初始化，在使用binder的过程中，从来没有执行过new Binder的方式来实现Binder初始化，原因很简单：binder初始化有它自身独立的特点。

每一个应用进程启动的时候，都是通过zygote fork产生的，所以，当fork产生进程后app进程的代码就开始执行，开始运行的地方如下：

```java
public static final Runnable zygoteInit(int targetSdkVersion, 
                                            long[] disabledCompatChanges,
                                            String[] argv, ClassLoader classLoader) {
		RuntimeInit.redirectLogStreams();
        RuntimeInit.commonInit();//初始化运行环境 
        ZygoteInit.nativeZygoteInit();//启动Binder ，方法在 androidRuntime.cpp中注册
        // 通过反射创建程序入口函数的 Method 对象，并返回 Runnable 对象
        //ActivityThread.main();
        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
                classLoader);
    }
```

可以看到，会执行ZygoteInit.nativeZygoteInit()函数，而nativeZygoteInit函数执行appRuntime的onZygoteInit
代码，也就是App_main.cpp中的 onZygoteInit()函数，函数如下：

```c++
virtual void onZygoteInit()
{
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool();
}
```

在ProcessState的self函数里面就会初始化ProcessState（），而这个初始化的一个非常重要的动作就是启动binder驱动和并构建binder的Map映射。具体代码如下：

```c++
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver)) //打开binder的虚拟驱动
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mBinderContextCheckFunc(nullptr)
    , mBinderContextUserData(nullptr)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{
    if (mDriverFD >= 0) {
        //调用mmap接口向Binder驱动中申请内核空间的内存
        // mmap the binder, providing a chunk of virtual address space to receive  transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
}
```

所以，总的来说，Binder的初始化是在进程已创建就完成了。创建进程后会第一时间为这个进程打开一个binder驱动，并调用mmap接口向Binder驱动中申请内核空间的内存。

#### Binder通信的流程 

1. 首先，一个进程使用 BINDERSETCONTEXT_MGR 命令通过 Binder 驱动将自己注册成为 ServiceManager，Binder的引用在所有Client中都固定为0；

2. Server 通过驱动向 ServiceManager 中注册 Binder（Server 中的 Binder 实体），表明可以对外提供服务。驱动为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager，ServiceManger 将其填入查找表。

3. Client 通过名字，在 Binder 驱动的帮助下从 ServiceManager 中获取到对 Binder 实体的引用。

4. 通过这个Binder实体引用，Client实现和 Server 进程的通信。

#### aidl通信的基本步骤

   1. Client通过ServiceConnection获取到Server的Binder，并且封装成一个Proxy。 

   2. 通过Proxy来同步调用IPC方法（xxxFunction），同时通过Parcel将参数传给Binder，最终触发Binder的 transact方法。

   3. Binder的transact方法最终会触发到Server上Stub的onTransact方法。 

   4. Server上Stub的onTransact方法中，会先从Parcel中解析中参数，然后将参数带入真正的方法中执行，然后将 结果写入Parcel后传回。 

   5. 请注意：Client的Ipc方法中，执行Binder的transact时，是阻塞等待的，一直到Server逻辑执行结束后才会继 续执行。当然，如果IPC方法是oneWay的方式，那么就是非阻塞的等待。 

   6. 当Server返回结果后，Client从Parcel中取出返回值，于是实现了一次IPC调用。

   #### bindService的全流程

   将Server端的Binder对象发送给Client端

   1. Activity作为Client发起bindService，最终会调度到AMS 去执行bindService。在这个过程中，Client要去调用 AMS的代码，所以此时就会涉及到跨进程调度，基于第三章的Binder通信模型我们不难知道，Client会先和 ServiceManager通信，从ServiceManager中拿到AMS的IBinder。 
   2. Activity拿到AMS的IBinder后，跨进程执行AMS的BindService函数； 
   3. 由于AMS管理所有的应用进程，因此AMS中持有了应用进程的Binder，所以此时AMS可以发起第4步也就是跨进 程调度scheduleBindService(); 
   4. Server端会在收到AMS的bindService的请求后，会将自己的IBinder发送给client，但是Server必须通过AMS才能 将Binder对象传过去，所以此时需要跨进程从ServiceManager中去拿到AMS的binder； 
   5. Server端通过AMS的binder直接调用AMS的代码publishService(),将service的Binder发送给AMS； 
   6. 经过层层调用，最终AMS讲Server端的binder通过回调connect函数传递给了Client端的Activity；

#### Java与Native 通信的基本流程 

![](https://note.youdao.com/yws/api/personal/file/WEB952bd9647a30851bd833b99e2e0597b6?method=download&shareKey=78cc7149cb6cf7706498543e988d821b)

1. Client端通过ServiceManager 拿到Server端服务的Binder 代理，也就是BinderProxy(是Server端Binder的一个代理)；
2. 这个BinderProxy的访问需要经过JNI层的Android_util_binder类将请求转交给native的BpBinder（p代表代理的意思）；
3. BpBinder会通过ioctl将请求转交给Binder驱动设备；
4. 在服务端注册了一个监听请求的回调函数，一旦驱动层收到BpBinder 的调用，就会回调BBInder注册的回调函数，于是，就将请求转给了BBinder；
5. BBinder拿到请求后，会进行一些数据的处理，然后通过JNI将请求转交给了java类；
6. java层会通过aidl中的函数将请求发送给Server端的实现者，由Server端通过stub 去调用相关的执行代码，并将结果通过类似的路径返回。

#### ServcieManager场景

在跨进程通信过程中，比如与AMS通信，首先需要拿到AMS的binder，然而，AMS的binder往往是通过ServcieManager获取的，因此会有代码：

```java
ServiceManager.getService(Context.ACTIVITY_SERVICE) 
```

j接下来调用Binder.allowBlocking(rawGetService(name))，核心代码在rawGetService中

```java
private static IBinder rawGetService(String name) throws RemoteException { 
 ...
 final IBinder binder = getIServiceManager().getService(name); 
 ...
 return binder; 
}
```

这里的逻辑是，先通过getIServiceManager()获取到IServiceManager对象，然后再通过这个IServiceManager对象获取到一个IBinder。
注意，此处有两个IPC：

1. 获取到IServiceManager

2. 通过IServiceManager获取到该name的Service的Binder的IPC

  重在探索java到native的逻辑，先看第一个。

```java
private static IServiceManager getIServiceManager() { 
 if (sServiceManager != null) { 
 return sServiceManager; 
 } 
 // Find the service manager 
 sServiceManager = ServiceManagerNative 
 .asInterface(Binder.allowBlocking(BinderInternal.getContextObject())); 
 return sServiceManager; 
} 
```

到这，可以看到返回的其实是ServiceManagerNative.asInterface()的返回值。
对AIDL有所了解就知道，asInterface()的内容大概如下：
这个方法属于aidl接口的内部类 Stub。 在同一进程中，就会直接返回Stub，如果在另一个进程中调用，就会返回将这个ibinder封装好的Proxy对象。后面会分析ServiceManagerNative.asInterface 函数。先看一下IBinder的来源，也就BinderInternal.getContextObject()。
BinderInternal.getContextObject()代码如下：

```java
public static final native IBinder getContextObject(); 
```

JNI层代码如下：
android_util_Binder.android_os_BinderInternal_getContextObject()代码如下

```c++
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz) 
{ 
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL); 
    return javaObjectForIBinder(env, b); 
} 
```

先通过ProcessState获取到了一个native 层的IBinder强引用，也就是一个BpBinder。 然后将这个native层的IBinder强引用传入javaObjectForIBinder()方法，最终封装成java层的IBinder然后返回。此处先不深究ProcessState的逻辑，整个native层的binder有自己的一整套的逻辑，后面的文章会继续探索。我们可以先稍微看下javaObjectForIBinder()的大概逻辑。 android_util_Binder.javaObjectForIBinder()

```c++
// 将一个BpBinder对象(这是native中的类型)转换成java中的类型 
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val) 
{  
 //JavaBBinder返回true，其他类均返回flase 
    if (val->checkSubclass(&gBinderOffsets)) { 
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object. 
        jobject object = static_cast<JavaBBinder*>(val.get())->object(); 
        LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object); 
        return object; 
    } 
 
    BinderProxyNativeData* nativeData = new BinderProxyNativeData(); 
    nativeData->mOrgue = new DeathRecipientList; 
    nativeData->mObject = val; 
 
    // 核心代码：运用反射创建一个BinderProxy对象 
    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass, 
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get()); 
    if (env->ExceptionCheck()) { 
        // In the exception case, getInstance still took ownership of nativeData. 
        return NULL; 
    } 
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object); 
 	//如果object是刚刚新建出来的BinderProxy 
    if (actualNativeData == nativeData) { 
 		//处理proxy计数 
        // Created a new Proxy 
        uint32_t numProxies = gNumProxies.fetch_add(1, std::memory_order_relaxed); 
        uint32_t numLastWarned = gProxiesWarned.load(std::memory_order_relaxed); 
        if (numProxies >= numLastWarned + PROXY_WARN_INTERVAL) { 
            // Multiple threads can get here, make sure only one of them gets to 
            // update the warn counter. 


            if (gProxiesWarned.compare_exchange_strong(numLastWarned, 
                        numLastWarned + PROXY_WARN_INTERVAL, std::memory_order_relaxed)) 
            { 
                ALOGW("Unexpectedly many live BinderProxies: %d\n", numProxies); 
            } 
        } 
    } else { 
        delete nativeData; 
    } 
 
    return object;  //object 是反射参数的java的 BinderProxy 
} 
```

上面的函数就是将一个BpBinder对象(这是native中的类型)转换成java中的类型，中间采用了反射技术而已。

核心代码gBinderProxyOffsets

```c++
static struct binderproxy_offsets_t 
{ 
    // Class state. 
    jclass mClass; 
    jmethodID mGetInstance; 
    jmethodID mSendDeathNotice; 
 
    // Object state. 
    //指向BinderProxyNativeData的指针 
    jfieldID mNativeData;  // Field holds native pointer to BinderProxyNativeData. 
} gBinderProxyOffsets; 
 
const char* const kBinderProxyPathName = "android/os/BinderProxy"; 
 
static int int_register_android_os_BinderProxy(JNIEnv* env) 
{ 
    ... 
    jclass clazz = FindClassOrDie(env, kBinderProxyPathName); 
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz); 
    gBinderProxyOffsets.mGetInstance = GetStaticMethodIDOrDie(env, clazz, "getInstance", 
            "(JJ)Landroid/os/BinderProxy;"); 
    gBinderProxyOffsets.mSendDeathNotice = 
            GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice", 
                                   "
(Landroid/os/IBinder$DeathRecipient;Landroid/os/IBinder;)V"); 
    gBinderProxyOffsets.mNativeData = GetFieldIDOrDie(env, clazz, "mNativeData", "J"); 
    ... 
} 
```

gBinderProxyOffsets实际上是一个用来记录一些java中对应类、方法以及字段的结构体，用于从native层调用java层代码，而通过int_register_android_os_BinderProxy，我们知道，binderproxy_oﬀsets_t中的mClass字段就是 BInderProxy，而mGetInstance 就是BInderProxy.java 中getInstance方法。因此核心代码创建的是一个BinderProxy对象。

具体的执行流程如下图所示：

![](https://note.youdao.com/yws/api/personal/file/WEBe8b639edcfbc0ca9f2ce30e8c7ae5dff?method=download&shareKey=ac21ff3e14c9010a0b4976ceea2e6d25)

**第一大步**：为了获取ServcieManager 的IServiceManager，首先要ServcieManager进程创建一个底层的Binder，所
以会有android_os_BinderInternal_getContextObject也就是第2步，第2步会在ProcessState::self()中初始化Binder
驱动，然后再执行第3步；
**第二大步**：上图中第3步会调用到第4步，在第4步getStrongProxyForHandle代码如下：

```c++
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle) 
{ 
    sp<IBinder> result; 
 
    AutoMutex _l(mLock); 
 
 //查找或建立handle对应的handle_entry 
    handle_entry* e = lookupHandleLocked(handle); 
 
    if (e != nullptr) { 
        IBinder* b = e->binder; 
        if (b == nullptr || !e->refs->attemptIncWeak(this)) { 
            if (handle == 0) { 
			 //当handle为ServiceManager的特殊情况 
			 //需要确保在创建Binder引用之前，context manager已经被binder注册 
			 //需要先确保ServcieManager活着 
                Parcel data; 
                status_t status = IPCThreadState::self()->transact( 
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0); 
                if (status == DEAD_OBJECT) 
                   return nullptr; 
            } 
 
 			//创建BpBinder并保存下来以便后面再次查找 
            b = BpBinder::create(handle); 
            e->binder = b; 
            if (b) e->refs = b->getWeakRefs(); 
            result = b; 
        } else { 
            // This little bit of nastyness is to allow us to add a primary 
            // reference to the remote proxy when this team doesn't have one 
            // but another team is sending the handle to us. 
            result.force_set(b); 
            e->refs->decWeak(this); 
        } 
    } 
    return result; 
} 
```

其中会执行一个BpBinder::create(handle)，此处会创建一个BpBinder对象，并且将BpBinder对象赋值给result对象
并返回result，也就是返回BpBinder对象。这个BpBinder对象一路返回，最终在android_util_Binder.cpp中的
android_os_BinderInternal_getContextObject函数中执行。

**第三大步**：BpBinder需要返回给java层Client端使用，所以此时的封装就是将BpBinder封装成为Java层的BinderProxy对象。因此在Client端得到的IServiceManager 其实是BinderProxy类的子类的对象。所以，BinderInternal.getContextObject()，返回的是一个层层封装的类的实例，具体来说，是Native层的BpBinder对象被封装成为BinderProxy对象并返回。

#### 不同类型的Binder

##### IBinder 

```c++
// IBinder从Refbase继承而来，一提供强弱指针计数能力 
class IBinder : public virtual RefBase 
{ 
public: 
    enum { 
        FIRST_CALL_TRANSACTION  = 0x00000001, 
        LAST_CALL_TRANSACTION   = 0x00ffffff, 
        PING_TRANSACTION        = B_PACK_CHARS('_','P','N','G'), 
        DUMP_TRANSACTION        = B_PACK_CHARS('_','D','M','P'), 
        INTERFACE_TRANSACTION   = B_PACK_CHARS('_', 'N', 'T', 'F'), 
        SYSPROPS_TRANSACTION    = B_PACK_CHARS('_', 'S', 'P', 'R'), 
        // Corresponds to TF_ONE_WAY -- an asynchronous call. 
        FLAG_ONEWAY             = 0x00000001 
    }; 
 	IBinder(); 
 
    // 根据descriptor查询相应的IInterface对象 
    virtual sp<IInterface>  queryLocalInterface(const String16& descriptor); 
 
    // 获取descriptor描述符 
    virtual const String16& getInterfaceDescriptor() const = 0; 
 
    virtual bool            isBinderAlive() const = 0; 
    virtual status_t        pingBinder() = 0; 
    virtual status_t        dump(int fd, const Vector<String16>& args) = 0; 
 
    // transact binder通信函数 
    virtual status_t  transact(uint32_t code, 
                               const Parcel& data, 
                               Parcel* reply, 
							   uint32_t flags = 0) = 0; 
 
    // 死亡通知相应类 
    class DeathRecipient : public virtual RefBase 
    { 
    public: 
        virtual void binderDied(const wp<IBinder>& who) = 0; 
    }; 
     
    // 如其名，用于注册Binder用的 
 	virtual status_t  linkToDeath(const sp<DeathRecipient>& recipient, 
                                        void* cookie = NULL, 
                                        uint32_t flags = 0) = 0; 

    // 撤销用之前注册的死亡通知函数 
	virtual status_t  unlinkToDeath(  const wp<DeathRecipient>& recipient, 
                                            void* cookie = NULL, 
                                            uint32_t flags = 0, 
                                            wp<DeathRecipient>* outRecipient = NULL) = 0; 
 
    virtual bool heckSubclass(const void* subclassID) const; 
 
    typedef void (*object_cleanup_func)(const void* id, void* obj, void* cleanupCookie); 
 
    virtual void attachObject(   const void* objectID, 
                                            void* object, 
                                            void* cleanupCookie, 
                                            object_cleanup_func func) = 0; 
    virtual void* findObject(const void* objectID) const = 0; 
    virtual void  detachObject(const void* objectID) = 0; 
  
    // 返回服务端的binder引用 
virtual BBinder* localBinder(); 
    // 放回客户端的binder引用 
virtual BpBinder* remoteBinder(); 
 
protected: 
    virtual ~IBinder(); 
}; 
```

对于IBinder中的方法，基本都是没有实现的，这些方法的实现都交给继承它的子类来实现，那下面直接看BpBinder的内容。

##### BpBinder,BBinder

BpBinder和BBinder都是Android中与Binder通信相关的代表，他们都是从IBinder中继承而来的。其中BpBinder是客户端用来与Server交互的代理类，BBinder则是和proxy相对的一端，它是proxy交互的目的端。如果说Proxy代表客户端，那么BBinder就代表这服务端。这里BpBinder和BBinder是一一对应的，即某个BpBinder只能和对应的BBinder交互。

##### JavaBBinder

IBinder是BBinder的父类，BBinder是JavaBBinder的父类。
java层直接与native层交互的对象有两个——Binder对象与BinderProxy对象。
Binder对应“Binder在本进程”的场景，BinderProxy对应“Binder在其他进程”的场景。
native层javaBBinder与java层的Binder一一对应。
native层的BinderProxyNativeData与java层的BinderProxy一一对应。
在native层，gBinderProxyOﬀsets(binderproxy_oﬀsets_t)存储了java层binderProxy的对象与需要调用的方法和属
性。gBinderOﬀsets(binderproxy_oﬀsets_t)存储了java层binder的对象与需要调用的方法和属性。
ibinderForJavaObject负责通过java的Binder或者BinderProxy对象，找到并返回native层的IBinder对象。


javaObjectForIBinder通过native层的IBinder对象，找到或者封装成java对象返回。



#### ServiceManager.getService方法

第一步：先调用IServiceManager.Stub.Proxy的getService方法 

```java
public android.os.IBinder getService(java.lang.String name) throws 
android.os.RemoteException 
{ 
    android.os.Parcel _data = android.os.Parcel.obtain(); 
    android.os.Parcel _reply = android.os.Parcel.obtain(); 
    android.os.IBinder _result; 
    try { 
        _data.writeInterfaceToken(DESCRIPTOR); 
        _data.writeString(name); 
        boolean _status = mRemote.transact(Stub.TRANSACTION_getService, _data, _reply, 
                          0); 
        _reply.readException(); 
        _result = _reply.readStrongBinder(); 
    } 
    finally { 
        _reply.recycle(); 
        _data.recycle(); 
    } 
    return _result; 
} 
```

第二步：接下来会调用mRemote.transact 

而mRemote则是BinderProxy对象，所以接下来执行下面的代码：

```java
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws 
RemoteException { 
    //检查Parcel大小 if (CHECK_PARCEL_SIZE && parcel.dataSize() >= 800*1024)
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
    ... 
    //trace 
    ... 
    //Binder事务处理回调 
    ... 
    //AppOpsManager信息记录 
    ... 
    try { 
        final boolean result = transactNative(code, data, reply, flags); 
        if (reply != null && !warnOnBlocking) { 
            reply.addFlags(Parcel.FLAG_IS_REPLY_FROM_BLOCKING_ALLOWED_OBJECT); 
        } 
        return result; 
    } finally { 
        ... 
    } 
} 
```

首先，系统会通过checkParcel检测数据的格式和大小，Android默认设置了Parcel数据传输不能超过800k,如果超过了的话，便会调用Slog.wtfStack打印日志，需要注意的是，在当前进程不是系统进程并且系统也不是工程版本的情况下，这个方法是会结束进程的，所以在应用开发的时候，我们需要注意跨进程数据传输的大小，避免因此引发crash。

核心函数是调用transactNative方法，这是一个native方法，在frameworks/base/core/jni/android_util_Binder.cpp中实现。

```c++
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj, 
        jint code, jobject dataObj, jobject replyObj, jint flags)  
{ 
    if (dataObj == NULL) { 
        jniThrowNullPointerException(env, NULL); 
        return JNI_FALSE; 
    } 
 
    Parcel* data = parcelForJavaObject(env, dataObj); 
    if (data == NULL) { 
        return JNI_FALSE; 
    } 
    Parcel* reply = parcelForJavaObject(env, replyObj); 
    if (reply == NULL && replyObj != NULL) { 
        return JNI_FALSE; 
    } 
 
    IBinder* target = getBPNativeData(env, obj)->mObject.get(); 
    if (target == NULL) { 
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been 
finalized!"); 
        return JNI_FALSE; 
    } 
 
    //log 
    ... 
    status_t err = target->transact(code, *data, reply, flags); 
    //log 
    ... 
 
    if (err == NO_ERROR) { 
        return JNI_TRUE; 
    } else if (err == UNKNOWN_TRANSACTION) { 
        return JNI_FALSE; 
    } 
 
    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/,  


 data->dataSize()); 
    return JNI_FALSE; 
} 
```

这里首先是获得native层对应的Parcel并执行判断，Parcel实际上功能是在native中实现的，java中的Parcel类使用mNativePtr成员变量保存了其对应native中的Parcel的指针，然后调用getBPNativeData函数获得BinderProxy在native中对应的BinderProxyNativeData，再通过里面的mObject域成员变量得到其对应的BpBinder。

getBPNativeData(env, obj)->mObject.get();是上面代码的核心之一，调用getBPNativeData函数获得BinderProxy在native中对应的BinderProxyNativeData，target 事实上是BpBinder。

第三步，BpBinder 的transact函数 
此时就顺理成章的调用到了BpBinder的transact函数了

```c++
status_t BpBinder::transact( 
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) 
{ 
    // Once a binder has died, it will never come back to life. 
    //判断binder服务是否存活 
    if (mAlive) { 
        ... 
        status_t status = IPCThreadState::self()->transact( 
            mHandle, code, data, reply, flags); 
        if (status == DEAD_OBJECT) mAlive = 0; 
 
        return status; 
    } 
    return DEAD_OBJECT; 
} 
```

这里有一个Alive判断，可以避免对一个已经死亡的binder服务再发起事务，浪费资源，除此之外便是调用IPCThreadState的transact函数了。

第四步：IPCThreadState的transact函数调用 
路径：frameworks/native/libs/binder/IPCThreadState.cpp
ProcessState负责打开binder驱动并进行mmap映射，而IPCThreadState则是负责与binder驱动进行具体的交互
IPCThreadState也有一个self函数，与ProcessState的self不同的是，ProcessState是进程单例，而IPCThreadState是线程单例。
我们接着看它的ProcessState的transact函数:

```c++
status_t IPCThreadState::transact(int32_t handle, 
                                  uint32_t code, const Parcel& data, 
                                  Parcel* reply, uint32_t flags) 
{ 
    status_t err; 
    flags |= TF_ACCEPT_FDS; 
    //log 
    ...     
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr); 
    if (err != NO_ERROR) { 
        if (reply) reply->setError(err); 
        return (mLastError = err); 
    } 
    if ((flags & TF_ONE_WAY) == 0) {    //binder事务不为TF_ONE_WAY 
        //当线程限制binder事务不为TF_ONE_WAY时 
        if (UNLIKELY(mCallRestriction != ProcessState::CallRestriction::NONE)) { 
            if (mCallRestriction == ProcessState::CallRestriction::ERROR_IF_NOT_ONEWAY) { 
                //这个限制只是log记录 
                ALOGE("Process making non-oneway call (code: %u) but is restricted.", code); 
                CallStack::logStack("non-oneway call", CallStack::getCurrent(10).get(), 
                    ANDROID_LOG_ERROR); 
            } else /* FATAL_IF_NOT_ONEWAY */ { 
                //这个限制会终止线程 
                LOG_ALWAYS_FATAL("Process may not make non-oneway calls (code: %u).", code); 
            } 
        } 
        if (reply) { 
            err = waitForResponse(reply); 
        } else { 
            Parcel fakeReply; 
            err = waitForResponse(&fakeReply); 
        } 
        //log 
        ... 
    } else {  //binder事务为TF_ONE_WAY 
        err = waitForResponse(nullptr, nullptr); 
    } 
 
    return err; 
} 
```

这个函数的重点在于writeTransactionData和waitForResponse，我们依次分析:
首先看writeTransactionData:

```c++
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags, 
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer) 
{ 
    binder_transaction_data tr; 
 
    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */ 
    //目标binder句柄值，ServiceManager为0 
    tr.target.handle = handle; 
    tr.code = code; 
    tr.flags = binderFlags; 


    tr.cookie = 0; 
    tr.sender_pid = 0; 
    tr.sender_euid = 0; 
 
    const status_t err = data.errorCheck(); 
    if (err == NO_ERROR) { 
        //数据大小 
        tr.data_size = data.ipcDataSize(); 
        //数据区起始地址 
        tr.data.ptr.buffer = data.ipcData(); 
        //传递的偏移数组大小 
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t); 
        //偏移数组的起始地址 
        tr.data.ptr.offsets = data.ipcObjects(); 
    } else if (statusBuffer) { 
        tr.flags |= TF_STATUS_CODE; 
        *statusBuffer = err; 
        tr.data_size = sizeof(status_t); 
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer); 
        tr.offsets_size = 0; 
        tr.data.ptr.offsets = 0; 
    } else { 
        return (mLastError = err); 
    } 
    //核心代码所在 
    //这里为BC_TRANSACTION 
    mOut.writeInt32(cmd); 
    mOut.write(&tr, sizeof(tr)); 
 
    return NO_ERROR; 
} 
```

binder_transaction_data结构体(tr结构体）是向Binder驱动通信的数据结构，上面函数中，我们将binder请求码（这里为BC_TRANSACTION）和binder_transaction_data结构体依次写入到mOut中，为之后binder_tansaction做准备。
再看waitForResponse函数

```c++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult) 
{ 
    uint32_t cmd; 
    int32_t err; 
 
    while (1) { 
        if ((err=talkWithDriver()) < NO_ERROR) break; 
        err = mIn.errorCheck(); 
        if (err < NO_ERROR) break; 
        if (mIn.dataAvail() == 0) continue; 
        cmd = (uint32_t)mIn.readInt32(); 
        switch (cmd) { 
        case BR_ONEWAY_SPAM_SUSPECT: 
            ... 
        case BR_TRANSACTION_COMPLETE: 
            //当TF_ONE_WAY模式下收到BR_TRANSACTION_COMPLETE直接返回，本次binder通信结束 
            if (!reply && !acquireResult) goto finish; 
            break; 
        case BR_DEAD_REPLY: 
            ... 
        case BR_FAILED_REPLY: 
            ... 
        case BR_FROZEN_REPLY: 
            ... 
        case BR_ACQUIRE_RESULT: 
            ... 
        case BR_REPLY: 
            { 
                binder_transaction_data tr; 
                err = mIn.read(&tr, sizeof(tr)); 
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY"); 
                //失败直接返回 
                if (err != NO_ERROR) goto finish; 
 
                if (reply) {    //客户端需要接收replay 
                    if ((tr.flags & TF_STATUS_CODE) == 0) {    //正常reply内容 
                        reply->ipcSetDataReference( 
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer), 
                            tr.data_size, 
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets), 
                            tr.offsets_size/sizeof(binder_size_t), 
                            freeBuffer /*释放缓冲区*/); 
                    } else {    //内容只是一个32位的状态码 
                        //接收状态码 
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer); 
                        //释放缓冲区 
                        freeBuffer(nullptr, 
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer), 
                            tr.data_size, 
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets), 
                            tr.offsets_size/sizeof(binder_size_t)); 
                    } 
                } else {    //客户端不需要接收replay 
                    //释放缓冲区 
                    freeBuffer(nullptr, 
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer), 
                        tr.data_size, 
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets), 
                        tr.offsets_size/sizeof(binder_size_t)); 
                    continue; 
                } 
            } 
            goto finish; 
        default: 
            //这里是binder服务端部分的处理，现在不需要关注 
            err = executeCommand(cmd); 
            if (err != NO_ERROR) goto finish; 
            break; 
        } 
    } 
 
finish: 
    if (err != NO_ERROR) { 
        if (acquireResult) *acquireResult = err; 
        if (reply) reply->setError(err); 
        mLastError = err; 
        logExtendedError(); 
    } 
    return err; 
} 
```

这里有一个循环，正如函数名所描述，会一直等待到一整条binder事务链结束返回后才会退出这个循环，在这个循环的开头，便是talkWithDriver方法，在talkWithDriver 函数里面主要是调用Binder驱动的ioctl方法完成数据的传输。

上面整理了Client端通过ServiceManager的getService函数去获取对应服务的对象的过程，分析到了将getService请求发送给Binder驱动。下面继续开始分析服务端收到getService的请求后如何处理的总流程。

#### ServiceManager进程 死循环 

servicemanager进程的入口函数在frameworks\native\cmds\servicemanager\main.cpp中

```c++
int main(int argc, char** argv) { 
 //根据上面的rc文件，argc == 1, argv[0] == "/system/bin/servicemanager" 
    if (argc > 2) { 
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]"; 
    } 
 //此时，要使用的binder驱动为/dev/binder 
    const char* driver = argc == 2 ? argv[1] : "/dev/binder"; 
 
 //初始化binder驱动 
    sp<ProcessState> ps = ProcessState::initWithDriver(driver); 
    ps->setThreadPoolMaxThreadCount(0); 
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY); 
 
 //实例化ServiceManager，传入Access类用于鉴权 
    sp<ServiceManager> manager = new ServiceManager(std::make_unique<Access>()); 
    //将自身作为服务添加 
 if (!manager->addService("manager", manager, false /*allowIsolated*/, 
IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) { 
        LOG(ERROR) << "Could not self register servicemanager"; 
    } 
    //设置服务端Bbinder对象 
    IPCThreadState::self()->setTheContextObject(manager); 
 //设置成为binder驱动的context manager,成为上下文的管理者 
    ps->becomeContextManager(nullptr, nullptr); 
 
 //通过Looper epoll机制处理binder事务 
    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/); 
    //通知驱动BC_ENTER_LOOPER，监听驱动fd，有消息时回调到handleEvent处理binder调用 
    BinderCallback::setupTo(looper); 
    //服务的注册监听相关 
    ClientCallbackCallback::setupTo(looper, manager); 
    //无限循环等消息 
    while(true) { 
        looper->pollAll(-1);  
    } 
 
    // should not be reached 
    return EXIT_FAILURE; 
} 
```

这里的Looper和我们平常应用开发所说的Looper是一个东西，可以通过Looper::addFd函数监听文件描述符，通过Looper::pollAll或Looper::pollOnce函数接收消息，消息抵达后会回调LooperCallback::handleEvent函数

```c++
class BinderCallback : public LooperCallback { 
public: 
    static sp<BinderCallback> setupTo(const sp<Looper>& looper) { 
        sp<BinderCallback> cb = new BinderCallback; 
 
        int binder_fd = -1; 
        //向binder驱动发送BC_ENTER_LOOPER事务请求，并获得binder设备的文件描述符 
        IPCThreadState::self()->setupPolling(&binder_fd); 
        LOG_ALWAYS_FATAL_IF(binder_fd < 0, "Failed to setupPolling: %d", binder_fd); 
        // Flush after setupPolling(), to make sure the binder driver 
        // knows about this thread handling commands. 
        IPCThreadState::self()->flushCommands(); 
         //监听binder文件描述符 
        int ret = looper->addFd(binder_fd, 
                                Looper::POLL_CALLBACK, 
                                Looper::EVENT_INPUT, 
                                cb, 
                                nullptr /*data*/); 
        LOG_ALWAYS_FATAL_IF(ret != 1, "Failed to add binder FD to Looper"); 
        return cb; 
    } 
 
    int handleEvent(int /* fd */, int /* events */, void* /* data */) override { 
        //从binder驱动接收到消息并处理 
        IPCThreadState::self()->handlePolledCommands(); 
        return 1;  // Continue receiving callbacks. 
    } 
}; 
```

在servicemanager进程启动的过程中调用了BinderCallback::setupTo函数，这个函数首先向binder驱动发起了一个BC_ENTER_LOOPER事务请求，获得binder设备的文件描述符，然后调用Looper::addFd函数监听binder设备文件描述符，这样当binder驱动发来消息后，就可以通过Looper::handleEvent函数接收并处理了。

```c++
status_t IPCThreadState::setupPolling(int* fd) 
{ 
    if (mProcess->mDriverFD < 0) { 
        return -EBADF; 
    } 
    //设置binder请求码 
    mOut.writeInt32(BC_ENTER_LOOPER); 
    //检查写缓存是否有可写数据，有的话发送给binder驱动 
    flushCommands(); 
    //赋值binder驱动的文件描述符 
    *fd = mProcess->mDriverFD; 
    pthread_mutex_lock(&mProcess->mThreadCountLock); 
    mProcess->mCurrentThreads++; 
    pthread_mutex_unlock(&mProcess->mThreadCountLock); 
    return 0; 
} 
```

基于以上的分析，一旦Client端发送IPC请求，就会通过Binder驱动发送消息给服务端，而服务端则通过BinderCallback来接收消息，并做下一步的处理。

#### Binder驱动事务处理 

BinderCallback类重写了handleEvent函数，里面调用了IPCThreadState::handlePolledCommands函数来接收处理binder事务

```c++
status_t IPCThreadState::handlePolledCommands() 
{ 
    status_t result; 
    //当读缓存中数据未消费完时，持续循环 
    do { 
        result = getAndExecuteCommand(); 
    } while (mIn.dataPosition() < mIn.dataSize()); 
 
    //当我们清空执行完所有的命令后，最后处理BR_DECREFS和BR_RELEASE 
    processPendingDerefs(); 
    flushCommands(); 
    return result; 
} 
```

这个函数的重点在getAndExecuteCommand，首先无论如何从binder驱动那里读取并处理一次响应，如果处理完后发现读缓存中还有数据尚未消费完，继续循环这个处理过程（理论来说此时不会再从binder驱动那里读写数据，只会处理剩余读缓存）

```c++
status_t IPCThreadState::getAndExecuteCommand() 
{ 
    status_t result; 
    int32_t cmd; 
 
    //从binder驱动中读写数据（理论来说此时写缓存dataSize为0，也就是只读数据） 
    result = talkWithDriver(/* true */); 
    if (result >= NO_ERROR) { 
        size_t IN = mIn.dataAvail(); 
        if (IN < sizeof(int32_t)) return result; 
        //读取BR响应码 
        cmd = mIn.readInt32(); 
        ... 
        result = executeCommand(cmd); 
        ... 
    } 
    return result; 
} 
```

从binder驱动中读取数据，然后从数据中读取出BR响应码，接着调用executeCommand函数继续往下处理

```c++
status_t IPCThreadState::executeCommand(int32_t cmd) 
{ 
    BBinder* obj; 
    RefBase::weakref_type* refs; 
    status_t result = NO_ERROR; 
    switch ((uint32_t)cmd) { 
    ... 
    case BR_TRANSACTION_SEC_CTX: 
    case BR_TRANSACTION: 
        { 
            binder_transaction_data_secctx tr_secctx; 
            binder_transaction_data& tr = tr_secctx.transaction_data; 
            if (cmd == (int) BR_TRANSACTION_SEC_CTX) { 
                result = mIn.read(&tr_secctx, sizeof(tr_secctx)); 
            } else { 
                result = mIn.read(&tr, sizeof(tr)); 
                tr_secctx.secctx = 0; 
            } 
            ALOG_ASSERT(result == NO_ERROR, 
                "Not enough command data for brTRANSACTION"); 
            if (result != NO_ERROR) break; 
            //读取数据到缓冲区 
            Parcel buffer; 
            buffer.ipcSetDataReference( 
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer), 
                tr.data_size, 
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets), 
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this); 
            ... 
            Parcel reply; 
            status_t error; 
            //对于ServiceManager的binder节点来说，是没有ptr的 
            if (tr.target.ptr) { 
                // We only have a weak reference on the target object, so we must first try to 
                // safely acquire a strong reference before doing anything else with it. 
                //对于其他binder服务端来说，tr.cookie为本地BBinder对象指针 
                if (reinterpret_cast<RefBase::weakref_type*>( 
                        tr.target.ptr)->attemptIncStrong(this)) { 
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer, 
                            &reply, tr.flags); 
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this); 
                } else { 
                    error = UNKNOWN_TRANSACTION; 
                } 
            } else { 
                //对于ServiceManager来说，使用the_context_object这个BBinder对象 
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags); 
            } 
 
            if ((tr.flags & TF_ONE_WAY) == 0) { 
                LOG_ONEWAY("Sending reply to %d!", mCallingPid); 


                if (error < NO_ERROR) reply.setError(error); 
                //非TF_ONE_WAY模式下需要Reply 
                sendReply(reply, 0); 
            } else { 
                ... //TF_ONE_WAY模式下不需要Reply，这里只打了些日志 
            } 
            ... 
        } 
        break; 
        ... 
    } 
 
    if (result != NO_ERROR) { 
        mLastError = result; 
    } 
    return result; 
} 
```

重点分析这个函数在BR_TRANSACTION下的case，首先，这个函数从读缓存中读取了binder_transaction_data，我们知道这个结构体记录了实际数据的地址、大小等信息，然后实例化了一个Parcel对象作为缓冲区，从binder_transaction_data中将实际数据读取出来。
接着找到本地BBinder对象，对于ServiceManager来说就是之前在main函数中setTheContextObject的ServiceManager对象，而对于其他binder服务端来说，则是通过tr.cookie获取，然后调用BBinder的transact函数。

理论上来说，the_context_object->transact(tr.code, buﬀer, &reply, tr.ﬂags)，应该是执行ServiceManager中的transact函数，但是在ServiceManager中没有实现该函数，因此只能去父类BnServiceManager中去找transact函数，但是很不巧BnServiceManager中也不存在于是再找父类，只有在BBinder中存在transact函数，因此会执行到BBinder中的transact函数。

```c++
status_t BBinder::transact( 
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) 
{ 
    //确保从头开始读取数据 
    data.setDataPosition(0); 
    if (reply != nullptr && (flags & FLAG_CLEAR_BUF)) { 
        //标记这个Parcel在释放时需要将内存中数据用0覆盖（涉及安全） 
        reply->markSensitive(); 
    } 
    status_t err = NO_ERROR; 
    //这里的code是由binder客户端请求传递过来的 
    //是客户端与服务端的一个约定 
    //它标识了客户端像服务端发起的是哪种请求 
    switch (code) { 
        ... 
        default: 
            err = onTransact(code, data, reply, flags); 
            break; 
    } 
    // In case this is being transacted on in the same process. 
    if (reply != nullptr) { 
        //设置数据指针偏移为0，这样后续读取数据便会从头开始 
        reply->setDataPosition(0); 
        if (reply->dataSize() > LOG_REPLIES_OVER_SIZE) { 
            ALOGW("Large reply transaction of %zu bytes, interface descriptor %s, code %d", 
                  reply->dataSize(), String8(getInterfaceDescriptor()).c_str(), code); 
        } 
    } 
    return err; 
} 
```

分析下继承关系后，会执行到BnServiceManager中的onTransact中

```c++
android::status_t BnServiceManager::onTransact(uint32_t _aidl_code, const ::android::Parcel& 
_aidl_data, ::android::Parcel* _aidl_reply, uint32_t _aidl_flags) { 
  ::android::status_t _aidl_ret_status = ::android::OK; 
  switch (_aidl_code) { 
  // 省略代码 
  case ::android::IBinder::FIRST_CALL_TRANSACTION + 2 /* addService */: 
  { 
       ::android::binder::Status _aidl_status(addService(in_name, in_service,  in_allowIsolated, in_dumpPriority));  //addService 是核心 
        _aidl_ret_status = _aidl_status.writeToParcel(_aidl_reply); 
       if (((_aidl_ret_status) != (::android::OK))) { 
      break; 
    	} 
      if (!_aidl_status.isOk()) { 
      	break; 
    	} 
 	 }
   break; 
   default: 
  { 
    _aidl_ret_status = ::android::BBinder::onTransact(_aidl_code, _aidl_data, _aidl_reply, 
_aidl_flags); 
  }
  return _aidl_ret_status;
}
```

代码的核心是addService，而addService的实现类就是ServiceManager里面的addService函数

```c++
Status ServiceManager::addService(const std::string& name, const sp<IBinder>& binder, bool 
allowIsolated, int32_t dumpPriority) { 
    auto ctx = mAccess->getCallingContext(); 
    // apps cannot add services 
#ifndef VENDORSERVICEMANAGER 
    if (!meetsDeclarationRequirements(binder, name)) { 
        // already logged 
        return Status::fromExceptionCode(Status::EX_ILLEGAL_ARGUMENT); 
    } 
#endif  // !VENDORSERVICEMANAGER 
 
    // implicitly unlinked when the binder is removed 
    if (binder->remoteBinder() != nullptr && binder->linkToDeath(this) != OK) { 
        LOG(ERROR) << "Could not linkToDeath when adding " << name; 
        return Status::fromExceptionCode(Status::EX_ILLEGAL_STATE); 
    } 
 
    // Overwrite the old service if it exists 
    mNameToService[name] = Service { 
        .binder = binder, 
        .allowIsolated = allowIsolated, 
        .dumpPriority = dumpPriority, 
        .debugPid = ctx.debugPid, 
    }; 
 
    auto it = mNameToRegistrationCallback.find(name); 
    if (it != mNameToRegistrationCallback.end()) { 
        for (const sp<IServiceCallback>& cb : it->second) { 
            mNameToService[name].guaranteeClient = true; 
            // permission checked in registerForNotifications 
            cb->onRegistration(name, binder); 
        } 
    } 
    return Status::ok(); 
} 
```

最终，服务的binder被封装成为了一个Service，并复制给了mNameToService数组进行存储。

#### IPCThreadState 解析 

在Android中，每个参与Binder通信的线程都会有一个IPCThreadState实例与之关联。我最开始接触到这个类是在BpBinder::transact方法中。

```c++
tatus_t BpBinder::transact( 
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) 
{ 
    // Once a binder has died, it will never come back to life. 
    //判断binder服务是否存活 
    if (mAlive) { 
       //非核心代码 
        status_t status = IPCThreadState::self()->transact( 
            mHandle, code, data, reply, flags); 
        if (status == DEAD_OBJECT) mAlive = 0; 
 
        return status; 
    } 
    return DEAD_OBJECT; 
} 
```

其就是调用的IPCThreadState::transact来完成的数据传输工作，其工作可以分为两步：

1、发送数据 

实际上，writeTransactionData只是将数据转换成binder_transaction_data结构并重新写入到IPCThreadState::mOut中。
并没有真正的将数据发送出去。实际的发送操作是在waitForResponse中完成的。

2、接收数据 

F_ONE_WAY表示的是单向通信，不需要对端回复。所以这里接收数据就多了几个判断分支。区别就是参数不一样。
该函数必定需要被执行的，因为数据要发出去

```c++
status_t IPCThreadState::transact(int32_t handle, 
                                  uint32_t code, const Parcel& data, 
                                  Parcel* reply, uint32_t flags) 
{ 
    status_t err; 
    flags |= TF_ACCEPT_FDS; 
 
 //writeTransactionData函数用于传输数据，其中第一个参数BC_TRANSACTION 
 //代表向Binder驱动发送命令协议，向Binder设备发送的命令协议都以BC_开头， 
 //而Binder驱动返回的命令协议以BR_开头 
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr); 
 
    if (err != NO_ERROR) { 
        if (reply) reply->setError(err); 
        return (mLastError = err); 
    } 
 
    if ((flags & TF_ONE_WAY) == 0) {  //binder事务不为TF_ONE_WAY 
        //省略代码 
        if (reply) { 
            err = waitForResponse(reply); 
        } else { 
            Parcel fakeReply; 
            err = waitForResponse(&fakeReply); 
        } 
 
    } else { 
        err = waitForResponse(nullptr, nullptr); 
    } 
    return err; 
} 
```

```c++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult) 
{ 
    uint32_t cmd; 
    int32_t err; 
    while (1) { 
        if ((err=talkWithDriver()) < NO_ERROR) break; 
        err = mIn.errorCheck(); 
        if (err < NO_ERROR) break; 
        if (mIn.dataAvail() == 0) continue; 
        cmd = (uint32_t)mIn.readInt32(); 
 
        IF_LOG_COMMANDS() { 
            alog << "Processing waitForResponse Command: " 
                << getReturnString(cmd) << endl; 
        } 
 
        switch (cmd) { 
        //处理命令 
 
        default: 
 			//这里是binder服务端部分的处理
            err = executeCommand(cmd); 
            if (err != NO_ERROR) goto finish; 
            break; 
        } 
    } 
 
finish: 
    if (err != NO_ERROR) { 
        if (acquireResult) *acquireResult = err; 
        if (reply) reply->setError(err); 
        mLastError = err; 
    } 
 
    return err; 
} 
```

需要处理如下这些cmd

```c++
static const char *kReturnStrings[] = { 
    "BR_ERROR", 
    "BR_OK", 
    "BR_TRANSACTION", 
    "BR_REPLY", 
    "BR_ACQUIRE_RESULT", 
    "BR_DEAD_REPLY", 
    "BR_TRANSACTION_COMPLETE", 
    "BR_INCREFS", 
    "BR_ACQUIRE", 
    "BR_RELEASE", 
    "BR_DECREFS", 
    "BR_ATTEMPT_ACQUIRE", 
    "BR_NOOP", 
    "BR_SPAWN_LOOPER", 
    "BR_FINISHED", 
    "BR_DEAD_BINDER", 
    "BR_CLEAR_DEATH_NOTIFICATION_DONE", 
    "BR_FAILED_REPLY", 
    "BR_TRANSACTION_SEC_CTX", 
}; 
```

处理CMD的函数就是waitForResponse和executeCommand。

前面通过writeTransactionData,已经把数据写入到了binder_transaction_data中。
talkWithDriver就是调用ioctl(BINDER_WRITE_READ)完成真正的数据接发

```c++
status_t IPCThreadState::talkWithDriver(bool doReceive) 
{ 
 	//检查打开的binder设备的fd 
    if (mProcess->mDriverFD < 0) { 
        return -EBADF; 
    } 
 
    binder_write_read bwr; 
 
    // Is the read buffer empty? 
    const bool needRead = mIn.dataPosition() >= mIn.dataSize(); 
 
    //需要写的数据大小，这里的doReceive默认为true，如果上一次的数据还没读完，则不会写入任何内容 
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0; 
 
    bwr.write_size = outAvail; 
    bwr.write_buffer = (uintptr_t)mOut.data(); 
 
    // This is what we'll read. 
    if (doReceive && needRead) { 
 		//将read_size设置为读缓存可用容量 
        bwr.read_size = mIn.dataCapacity(); 
 		//设置读缓存起始地址 
        bwr.read_buffer = (uintptr_t)mIn.data(); 
    } else { 
        bwr.read_size = 0; 
        bwr.read_buffer = 0; 
    } 
    // Return immediately if there is nothing to do. 
     //没有要读写的数据就直接返回 
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR; 
 
    bwr.write_consumed = 0; 
    bwr.read_consumed = 0; 
    status_t err; 
    //省略异常处理log 
    if (err >= NO_ERROR) { 
 		//写数据被消费了 
        if (bwr.write_consumed > 0) { 
			 //写数据没有被消费完 
            if (bwr.write_consumed < mOut.dataSize()) 
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. " 
                                 "err: %s consumed: %zu of %zu", 
                                 statusToString(err).c_str(), 
                                 (size_t)bwr.write_consumed, 
                                 mOut.dataSize()); 
            else { 
 				//写数据消费完了，将数据大小设置为0，这样下次就不会再写数据了 
                mOut.setDataSize(0); 
                processPostWriteDerefs(); 
            } 
        } 
 		//读到了数据 
        if (bwr.read_consumed > 0) { 
			 //设置数据大小及数据指针偏移，这样后面就可以从中读取出来数据了 
            mIn.setDataSize(bwr.read_consumed); 
            mIn.setDataPosition(0); 
        } 
        return NO_ERROR; 
    } 
 
    return err; 
} 
```

