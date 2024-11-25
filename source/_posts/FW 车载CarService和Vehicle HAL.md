---
title: Android 13 车载CarService和Vehicle HAL
date: 2024-10-24 22:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

车载 HAL 与 Android Automotive 架构

- **Car API**：内有包含 `CarSensorManager` 在内的 API。如需详细了解受支持的 API，请参阅 `/platform/packages/services/Car/car-lib`。
- **CarService**：位于 `/platform/packages/services/Car/`。
- **车载 HAL**：用于定义 OEM 可以实现的车辆属性的接口。包含属性元数据（例如，车辆属性是否为 int 以及允许使用哪些更改模式）。位于 `hardware/libhardware/include/hardware/vehicle.h`。如需了解基本参考实现，请参阅 `hardware/libhardware/modules/vehicle/`。

[Automotive  | Android Open Source Project](https://source.android.com/docs/devices/automotive?hl=zh-cn)

[汽车框架核心  | Android Open Source Project](https://source.android.google.cn/docs/automotive/car-framework-core?hl=zh-cn)

#### 汽车框架核心

![](https://note.youdao.com/yws/api/personal/file/WEBa1269c3146640700735649e3ec5b6fe4?method=download&shareKey=285f6480c42d8b10c78d4b620b02921c)

传统 Android 堆栈和汽车堆栈之间的区别：
● 汽车专用应用使用汽车 API 来访问汽车服务实现的功能。
● 汽车服务通过 CarServiceHelperService 与系统服务器通信，以访问核心 Android 功能。
● CarServiceHelperService 的主要目的是启动汽车服务。但是，当没有指定 API 与系统服务器通信时，将使用 CSHS。
● 汽车服务还连接到汽车专用的本机服务，例如 CarWatchdog。这些服务在系统服务器和汽车服务初始化之前处理汽车专用的任务。
● 使用车辆 HAL 和汽车专用 HAL 抽象出汽车硬件



![](https://note.youdao.com/yws/api/personal/file/WEB92b0a2491a0e312231c5bcd97f23cc32?method=download&shareKey=7d2d00f8036390b273f00d70d67049a8)

##### Car Service
● 汽车服务提供汽车特定功能和策略的实现。它允许应用程序与汽车硬件交互。
● 汽车服务 (com.android.car) 以用户 0 身份运行，可以为所有用户提供服务，无论用户切换如何。
● 另一个版本 (CarPerUserService) 为用户 10+ 运行
○ 这是因为蓝牙和 LocationManager 仅适用于当前用户。
○ 此服务仅供内部汽车服务使用，不直接提供任何汽车 API。
● 在客户端应用程序中创建 Car 对象时，它将包含一个 ICar Binder 对象，以便它可以与 Car Service 通信。
● ICarImpl 负责创建各种 CarXYZServices 并实现应用程序与之交互的 ICar.aidl 接口。

##### Car Managers
● 应用程序用来访问汽车服务功能的共享库。
● 公开汽车特定的 API 功能。
● 每个 CarXYZManager 都有一个关联的 CarXYZService对应项。
● CarXYZManagers 使用 AIDL 文件定义的 ICarXYZService 绑定接口与各个 CarXYZServices 进行通信。
● CarXYZServices 还与汽车服务内的 XYZHalServices 进行通信

使用Car APIs

```java
 private void init() {
  Car mCar = Car.createCar(mContext);
  (CarPropertyManager) mCarPropertyManager =
    mCar.getCarManager(CarPropertyManager.class);
 }
 private int getProperty(int propId, int areaId) {
  return mCarPropertyManager.getProperty(propId, areaId);
 }
```

监听CarService崩溃重新获取。

```java
private void init() {
  // This waits for the car service connection to be established.
  Car.createCar(mContext, null, CAR_WAIT_TIMEOUT_WAIT_FOREVER, 
this::onLifecycleChanged);
 }
 private void onLifecycleChanged(Car car, boolean ready) {
  synchronized (mLock) {
    if (ready) {
      mCarPropertyManager =
        (CarPropertyManager) car.getCarManager(CarPropertyManager.class);
      // Do initialization with CarXXXManager here, e.g. subscribe to 
property events.
    } else {
      Log.e(“Car service is disconnected”);
      mCarPropertyManager = null; 
      // Clear all internal state associated with CarXXXManager.
    }
  }
 }
```

#### CarAPI

packages/services/Car/car-lib/  aidl接口，客户端IBinder对象在Car API中被封装在一个个`XXXManager`类中，列举其中几个

- CarActivityManager：运行Task状态监听，设置
- CarDrivingStateManager：车辆运动状态监听
- CarAudioManager：声音相关
- CarMediaManager：多媒体播放源相关
- CarPropertyManager：车辆相关，空调车门车窗等数据设置获取和监听

#### CarService启动

##### 启动CarServiceHelperService

```java
 private static final String CAR_SERVICE_HELPER_SERVICE_CLASS =
            "com.android.internal.car.CarServiceHelperService";
private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
 	boolean isAutomotive = mPackageManager
                    .hasSystemFeature(PackageManager.FEATURE_AUTOMOTIVE);
            if (isAutomotive) {
                t.traceBegin("StartCarServiceHelperService");
                final SystemService cshs = mSystemServiceManager
                        .startService(CAR_SERVICE_HELPER_SERVICE_CLASS);
                ...
}
```

CarServiceHelperService中执行onStart

```java
//frameworks/opt/car/services/builtInServices/src/com/android/internal/car/CarServiceHelperService.java
private static final String CSHS_UPDATABLE_CLASSNAME_STRING =
            "com.android.internal.car.updatable.CarServiceHelperServiceUpdatableImpl";
CarServiceHelperService(
            Context context,
            CarLaunchParamsModifier carLaunchParamsModifier,
            CarWatchdogDaemonHelper carWatchdogDaemonHelper,
            @Nullable CarServiceHelperServiceUpdatable carServiceHelperServiceUpdatable,
            @Nullable CarDevicePolicySafetyChecker carDevicePolicySafetyChecker) {
    //创建CarServiceHelperServiceUpdatableImpl
     mCarServiceHelperServiceUpdatable = (CarServiceHelperServiceUpdatable) Class
                        .forName(CSHS_UPDATABLE_CLASSNAME_STRING)
                        .getConstructor(Context.class, CarServiceHelperInterface.class,
                                CarLaunchParamsModifierInterface.class)
                        .newInstance(mContext, this,
                                mCarLaunchParamsModifier.getBuiltinInterface());
 }
@Override
    public void onStart() {
        ...
        mCarServiceHelperServiceUpdatable.onStart();
    }

//frameworks/opt/car/services/updatableServices/src/com/android/internal/car/updatable/CarServiceHelperServiceUpdatableImpl.java
    @Override
    public void onStart() {
        //bind这个服务action = android.car.ICar
        Intent intent = new Intent(CAR_SERVICE_INTERFACE).setPackage(CAR_SERVICE_PACKAGE);
        Context userContext = mContext.createContextAsUser(UserHandle.SYSTEM, /* flags= */ 0);
        //Connection返回的IBinder mCarServiceBinder = ICar.Stub.asInterface(iBinder);
        if (!userContext.bindService(intent, Context.BIND_AUTO_CREATE, this,
                mCarServiceConnection)) {
            Slogf.wtf(TAG, "cannot start car service");
        }
    }
```

##### 启动CarService

```java
//packages/services/Car/service-builtin/src/com/android/car/CarService.java
public class CarService extends ServiceProxy {
    public CarService() {
        //CAR_SERVICE_IMPL_CLASS = "com.android.car.CarServiceImpl"
        super(UpdatablePackageDependency.CAR_SERVICE_IMPL_CLASS);
    }
}
public class CarServiceImpl extends ProxiedService {
     @Override
    public void onCreate() {
        //这个地方创建的是aidl实现的AidlVehicleStub（android.hardware.automotive.vehicle.IVehicle/default）
        //如果获取不到，返回hidl实现的HidlVehicleStub
        mVehicle = VehicleStub.newVehicleStub();
        mVehicleInterfaceName = mVehicle.getInterfaceDescriptor();
		
        mICarImpl = new ICarImpl(this,
                getBuiltinPackageContext(),
                mVehicle,
                SystemInterface.Builder.defaultSystemInterface(this).build(),
                mVehicleInterfaceName);
        mICarImpl.init();

        mVehicle.linkToDeath(mVehicleDeathRecipient);

        ServiceManagerHelper.addService("car_service", mICarImpl);
        SystemPropertiesHelper.set("boot.car_service_created", "1");
    }
}
```

##### ICarImpl创建

```java
ICarImpl(Context serviceContext, @Nullable Context builtinContext, VehicleStub vehicle,
            SystemInterface systemInterface, String vehicleInterfaceName,
            @Nullable CarUserService carUserService,
            @Nullable CarWatchdogService carWatchdogService,
            @Nullable CarPerformanceService carPerformanceService,
            @Nullable GarageModeService garageModeService,
            @Nullable ICarPowerPolicySystemNotification powerPolicyDaemon,
            @Nullable CarTelemetryService carTelemetryService) {
     mHal = constructWithTrace(t, VehicleHal.class,
                () -> new VehicleHal(serviceContext, vehicle));
     mCarPropertyService = constructWithTrace(
                t, CarPropertyService.class,
                () -> new CarPropertyService(serviceContext, mHal.getPropertyHal()));
     mCarDrivingStateService = constructWithTrace(
                t, CarDrivingStateService.class,
                () -> new CarDrivingStateService(serviceContext, mCarPropertyService));
        mCarOccupantZoneService = constructWithTrace(t, CarOccupantZoneService.class,
                () -> new CarOccupantZoneService(serviceContext));
    	...
}
```

CarService的服务已经启动，再看下CarPropertyService的具体流程。

#### CarPropertyService

##### 客户端CarPropertyManager

以获取VIN为例分析下流程

```java
services/Car/car-lib/src/android/car/VehiclePropertyIds.java
    /**
     * VIN of vehicle
     * Requires permission: {@link Car#PERMISSION_IDENTIFICATION}.
     */
    @RequiresPermission(Car.PERMISSION_IDENTIFICATION)
    @AddedInOrBefore(majorVersion = 33)
    public static final int INFO_VIN = 286261504; // 0x11100100

private void init() {
  Car mCar = Car.createCar(mContext);
  (CarPropertyManager) mCarPropertyManager =
    mCar.getCarManager(CarPropertyManager.class);
    mCarPropertyManager.getProperty(INFO_VIN,NO_AREA);
 }
 private int getProperty(int propId, int areaId) {
  return mCarPropertyManager.getProperty(propId, areaId);
 }
//客户端最终调用到这个方法，然后跨进程到SystemServer进程中CarService的子服务CarPropertyService中
private final ICarProperty mService;
public <E> CarPropertyValue<E> getProperty(int propId, int areaId) {
        checkSupportedProperty(propId);
            CarPropertyValue<E> propVal = mService.getProperty(propId, areaId);
            return propVal;
        }
```

##### 服务端代码

```java
//services/Car/service/src/com/android/car/CarPropertyService.java
public class CarPropertyService extends ICarProperty.Stub
        implements CarServiceBase, PropertyHalService.PropertyHalListener {
        @Override
    public CarPropertyValue getProperty(int prop, int zone)
            throws IllegalArgumentException, ServiceSpecificException {
        synchronized (mLock) {
            if (mConfigs.get(prop) == null) {
                // Do not attempt to register an invalid propId
                Slogf.e(TAG, "getProperty: propId is not in config list:0x" + toHexString(prop));
                return null;
            }
        }
         // Checks if android has permission to read property.
        String permission = mHal.getReadPermission(prop);
        CarServiceUtils.assertPermission(mContext, permission);
        //mHal需要确定下是哪个对象
        return mHal.getProperty(prop, zone);
    }
}
```

mHal是CarPropertyService创建的时候传递的参数

```java
public CarPropertyService(Context context, PropertyHalService hal) {
        if (DBG) {
            Slogf.d(TAG, "CarPropertyService started!");
        }
        mHal = hal;
        mContext = context;
    }

//ICarImpl中创建的CarPropertyService
mCarPropertyService = constructWithTrace(
                t, CarPropertyService.class,
                () -> new CarPropertyService(serviceContext, mHal.getPropertyHal()));

//ICarImpl中mHal
 mHal = constructWithTrace(t, VehicleHal.class,
                () -> new VehicleHal(serviceContext, vehicle));

//VehicleHal中的两个参数的构造函数
 public VehicleHal(Context context, VehicleStub vehicle) {
        this(context, /* powerHal= */ null, /* propertyHal= */ null,
                /* inputHal= */ null, /* vmsHal= */ null, /* userHal= */ null,
                /* diagnosticHal= */ null, /* clusterHalService= */ null,
                /* timeHalService= */ null, /* halClient= */ null,
                CarServiceUtils.getHandlerThread(VehicleHal.class.getSimpleName()), vehicle);
    }

 VehicleHal(Context context,
            PowerHalService powerHal,
            PropertyHalService propertyHal,
            InputHalService inputHal,
            VmsHalService vmsHal,
            UserHalService userHal,
            DiagnosticHalService diagnosticHal,
            ClusterHalService clusterHalService,
            TimeHalService timeHalService,
            HalClient halClient,
            HandlerThread handlerThread,
            VehicleStub vehicle) {
     	...
     	mPropertyHal = propertyHal != null ? propertyHal : new PropertyHalService(this);
     	// mPropertyHal must be the last so that on init/release
        // it can be used for all other HAL services properties.
        mHalClient = halClient != null
                ? halClient : new HalClient(vehicle, mHandlerThread.getLooper(),
                /* callback= */ this);
        mVehicleStub = vehicle;
 }
  public PropertyHalService getPropertyHal() {
      	//返回的是PropertyHalService(this)
        return mPropertyHal;
    }

//PropertyHalService.java
public CarPropertyValue getProperty(int mgrPropId, int areaId)
            throws IllegalArgumentException, ServiceSpecificException {
    //mVehicleHal在PropertyHalService创建传递过来，回到VehicleHal的get方法
     HalPropValue value = mVehicleHal.get(halPropId, areaId);
     HalPropConfig propConfig;
        synchronized (mLock) {
            propConfig = mHalPropIdToPropConfig.get(halPropId);
        }
        return value.toCarPropertyValue(mgrPropId, propConfig);
}

 public PropertyHalService(VehicleHal vehicleHal) {
        mPropIds = new PropertyHalServiceIds();
        mSubscribedHalPropIds = new HashSet<Integer>();
     	//VehicleHal对象，回到这个里面看下get方法
        mVehicleHal = vehicleHal;
    }
//VehicleHal中的get
public HalPropValue get(int propertyId, int areaId)
            throws IllegalArgumentException, ServiceSpecificException {
    	//VehicleHal构造函数mHalClient = new HalClient(vehicle, );
        return mHalClient.getValue(mPropValueBuilder.build(propertyId, areaId));
    }

//HalClient.java
 HalPropValue getValue(HalPropValue requestedPropValue)
            throws IllegalArgumentException, ServiceSpecificException {
        // Use a wrapper to create a final object passed to lambda.
        ObjectWrapper<ValueResult> resultWrapper = new ObjectWrapper<>();
        resultWrapper.object = new ValueResult();
        int status = invokeRetriable(() -> {
            resultWrapper.object = internalGet(requestedPropValue);
            return resultWrapper.object.status;
        }, mWaitCapMs, mSleepMs);

        ValueResult result = resultWrapper.object;
 }
	//调用HalClient的internalGet
	private ValueResult internalGet(HalPropValue requestedPropValue) {
        final ValueResult result = new ValueResult();
        	//mVehicle继续寻找这个参数
            result.propValue = mVehicle.get(requestedPropValue);
            result.status = StatusCode.OK;
            result.errorMsg = new String();
        return result;
    }
//mVehicle在HalClient构造函数中赋值，在VehicleHal中创建的
 HalClient(VehicleStub vehicle, Looper looper, HalClientCallback callback, int waitCapMs,
            int sleepMs) {
     	//mVehicle是CarServiceImpl#onCreate创建时mVehicle = VehicleStub.newVehicleStub()一路传递下去的
     	//
        mVehicle = vehicle;
 }

```

获取VIN的流程已经执行到了HalClient#internalGet中的mVehicle.get代码，mVehicle是VehicleStub的对象

```java
//VehicleStub.java
public static VehicleStub newVehicleStub() throws IllegalStateException {
        VehicleStub stub = new AidlVehicleStub();
        if (stub.isValid()) {
            return stub;
        }
    }
//AidlVehicleStub.java
private static final String AIDL_VHAL_SERVICE =
            "android.hardware.automotive.vehicle.IVehicle/default";
 AidlVehicleStub() {
        this(getAidlVehicle());
    }

AidlVehicleStub(IVehicle aidlVehicle) {
        mAidlVehicle = aidlVehicle;
        mPropValueBuilder = new HalPropValueBuilder(/*isAidl=*/true);
        mHandlerThread = CarServiceUtils.getHandlerThread(AidlVehicleStub.class.getSimpleName());
        mHandler = new Handler(mHandlerThread.getLooper());
        mGetSetValuesCallback = new GetSetValuesCallback();
 }

private static IVehicle getAidlVehicle() {
        try {
            //android.hardware.automotive.vehicle.IVehicle/default hal层的aidl服务
            return IVehicle.Stub.asInterface(
                    ServiceManagerHelper.waitForDeclaredService(AIDL_VHAL_SERVICE));
        } catch (RuntimeException e) {
            Slogf.w(CarLog.TAG_SERVICE, "Failed to get \"" + AIDL_VHAL_SERVICE + "\" service", e);
        }
        return null;
    }
```

看下AidlVehicleStub的get方法

```java
private final IVehicle mAidlVehicle;
public HalPropValue get(HalPropValue requestedPropValue)
            throws RemoteException, ServiceSpecificException {
       ...
    	//这个地方通过IVehicle获取数据，aidl接口
        mAidlVehicle.getValues(mGetSetValuesCallback, requests);
        AndroidAsyncFuture<GetValueResult> asyncResultFuture = new AndroidAsyncFuture(resultFuture);
           GetValueResult result = asyncResultFuture.get(mTimeoutMs, TimeUnit.MILLISECONDS);
         return mPropValueBuilder.build(result.prop);
       
    }
```

IVehicle.aidl在下面的目录中，这块属于hal层的vehicle代码，CarService通过Binder跨进程通信

hardware/interfaces/automotive/vehicle/aidl/aidl_api/android.hardware.automotive.vehicle/1/android/hardware/automotive/vehicle/IVehicle.aidl

aidl确实方便，比hidl少了好多步骤。在继续分析下Vehicle hal层的代码

#### Vehicle hal

hardware/libhardware/modules/vehicle/

在VehicleProperty.aidl中INFO_VIN跟VehiclePropertyIds.java的也是对应上的

```c++
//VehicleProperty
package android.hardware.automotive.vehicle;
@Backing(type="int") @VintfStability
enum VehicleProperty {
  INVALID = 0,
  INFO_VIN = 286261504,
 }
 //VehiclePropertyIds
 public static final int INFO_VIN = 286261504; // 0x11100100

//types.hal中的值也能对应上
    INFO_VIN = (
        0x0100
        | VehiclePropertyGroup:SYSTEM
        | VehiclePropertyType:STRING
        | VehicleArea:GLOBAL),
```

[HIDL  | Android Open Source Project](https://source.android.google.cn/docs/core/architecture/hidl?hl=zh-cn)

[AIDL 概览  | Android Open Source Project](https://source.android.google.cn/docs/core/architecture/aidl?hl=zh-cn)

服务启动

```c++
service vendor.vehicle-hal-default /vendor/bin/hw/android.hardware.automotive.vehicle@V1-default-service
    class early_hal
    user vehicle_network
    group system inet
```

服务声明

```java
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>android.hardware.automotive.vehicle</name>
        <version>1</version>
        <fqname>IVehicle/default</fqname>
    </hal>
</manifest>
```

Android.bp

```
cc_binary {
    name: "android.hardware.automotive.vehicle@V1-default-service",
    vendor: true,
    defaults: [
        "FakeVehicleHardwareDefaults",
        "VehicleHalDefaults",
        "android-automotive-large-parcelable-defaults",
    ],
    vintf_fragments: ["vhal-default-service.xml"],
    init_rc: ["vhal-default-service.rc"],
    relative_install_path: "hw",
    srcs: ["src/VehicleService.cpp"],
    static_libs: [
        "DefaultVehicleHal",
        "FakeVehicleHardware",
        "VehicleHalUtils",
    ],
    header_libs: [
        "IVehicleHardware",
    ],
    shared_libs: [
        "libbinder_ndk",
    ],
}
```

##### VehicleService.cpp

```C++
#include <DefaultVehicleHal.h>
#include <FakeVehicleHardware.h>

#include <android/binder_manager.h>
#include <android/binder_process.h>
sing ::android::hardware::automotive::vehicle::DefaultVehicleHal;
using ::android::hardware::automotive::vehicle::fake::FakeVehicleHardware;

int main(int /* argc */, char* /* argv */[]) {
    std::unique_ptr<FakeVehicleHardware> hardware = std::make_unique<FakeVehicleHardware>();
    std::shared_ptr<DefaultVehicleHal> vhal =
            ::ndk::SharedRefBase::make<DefaultVehicleHal>(std::move(hardware));
    //添加服务到ServiceManager
    binder_exception_t err = AServiceManager_addService(
            vhal->asBinder().get(), "android.hardware.automotive.vehicle.IVehicle/default");

    if (!ABinderProcess_setThreadPoolMaxThreadCount(4)) {
        ALOGE("%s", "failed to set thread pool max thread count");
        return 1;
    }
    ABinderProcess_startThreadPool();
    ALOGI("Vehicle Service Ready");
    ABinderProcess_joinThreadPool();
    return 0;
}
```

采用双层架构。上层是 `DefaultVehicleHal`，实现了 VHAL AIDL 接口，并提供适用于所有硬件设备的通用 VHAL 逻辑。下层是 `FakeVehicleHardware`，实现了 `IVehicleHardware` 接口。此类可模拟与实际硬件或车载总线交互的 VHAL 逻辑，并且特定于设备。供应商也可以视需要调整这一架构，重复使用同一 `DefaultVehicleHal` 类（扩展该类以覆盖某个方法），并提供自己的 `IVehicleHardware` 实现。

DefaultVehicleHal的构造函数

```C++
//hardware/interfaces/automotive/vehicle/aidl/impl/vhal/src/DefaultVehicleHal.cpp
class DefaultVehicleHal final : public aidl::android::hardware::automotive::vehicle::BnVehicle {
  DefaultVehicleHal::DefaultVehicleHal(std::unique_ptr<IVehicleHardware> hardware)
    : mVehicleHardware(std::move(hardware)),
      mPendingRequestPool(std::make_shared<PendingRequestPool>(TIMEOUT_IN_NANO)) {
    //获取所有属性配置
    auto configs = mVehicleHardware->getAllPropertyConfigs();
    for (auto& config : configs) {
        mConfigsByPropId[config.prop] = config;
    }
    VehiclePropConfigs vehiclePropConfigs;
    vehiclePropConfigs.payloads = std::move(configs);
    auto result = LargeParcelableBase::parcelableToStableLargeParcelable(vehiclePropConfigs);
    if (!result.ok()) {
        ALOGE("failed to convert configs to shared memory file, error: %s, code: %d",
              result.error().message().c_str(), static_cast<int>(result.error().code()));
        return;
    }

    if (result.value() != nullptr) {
        mConfigFile = std::move(result.value());
    }

    mSubscriptionClients = std::make_shared<SubscriptionClients>(mPendingRequestPool);
    mSubscriptionClients = std::make_shared<SubscriptionClients>(mPendingRequestPool);
	//subscribeIdByClient是一个线程安全类，用于维护每个订阅客户端的请求 ID 不断增加。此类可以安全地传递给异步回调。
    auto subscribeIdByClient = std::make_shared<SubscribeIdByClient>();
    IVehicleHardware* hardwarePtr = mVehicleHardware.get();
    mSubscriptionManager = std::make_shared<SubscriptionManager>(hardwarePtr);

    std::weak_ptr<SubscriptionManager> subscriptionManagerCopy = mSubscriptionManager;
    //注册一个回调，当车辆发生属性更改事件时将调用该回调，回调函数为onPropertyChangeEvent。
    mVehicleHardware->registerOnPropertyChangeEvent(
            std::make_unique<IVehicleHardware::PropertyChangeCallback>(
                    [subscriptionManagerCopy](std::vector<VehiclePropValue> updatedValues) {
                        onPropertyChangeEvent(subscriptionManagerCopy, updatedValues);
                    }));

    // Register heartbeat event.
    mRecurrentAction =
            std::make_shared<std::function<void()>>([hardwarePtr, subscriptionManagerCopy]() {
                checkHealth(hardwarePtr, subscriptionManagerCopy);
            });
    //注册心跳事件
    mRecurrentTimer.registerTimerCallback(HEART_BEAT_INTERVAL_IN_NANO, mRecurrentAction);

    mBinderImpl = std::make_unique<AIBinderImpl>();
    mOnBinderDiedUnlinkedHandlerThread = std::thread([this] { onBinderDiedUnlinkedHandler(); });
    mDeathRecipient = ScopedAIBinder_DeathRecipient(
            AIBinder_DeathRecipient_new(&DefaultVehicleHal::onBinderDied));
    AIBinder_DeathRecipient_setOnUnlinked(mDeathRecipient.get(),
                                          &DefaultVehicleHal::onBinderUnlinked);
}
}
```

BnVehicle是IVehicle.aidl的服务端实现，CarService远程调用到这个地方

看下客户端调用的mAidlVehicle.getValues(mGetSetValuesCallback, requests);对应这个方法

```C++
//hardware/interfaces/automotive/vehicle/aidl/impl/vhal/src/DefaultVehicleHal.cpp
ScopedAStatus DefaultVehicleHal::getValues(const CallbackType& callback,
                                           const GetValueRequests& requests) {
    expected<LargeParcelableBase::BorrowedOwnedObject<GetValueRequests>, ScopedAStatus>
            deserializedResults = fromStableLargeParcelable(requests);
    if (!deserializedResults.ok()) {
        ALOGE("getValues: failed to parse getValues requests");
        return std::move(deserializedResults.error());
    }
    const std::vector<GetValueRequest>& getValueRequests =
            deserializedResults.value().getObject()->payloads;
    // A list of failed result we already know before sending to hardware.
    //在发送到硬件之前,失败结果的列表
    std::vector<GetValueResult> failedResults;
    // The list of requests that we would send to hardware.
    //发送给硬件的请求列表
    std::vector<GetValueRequest> hardwareRequests;

    for (const auto& request : getValueRequests) {
        if (auto result = checkReadPermission(request.prop); !result.ok()) {
            ALOGW("property does not support reading: %s", getErrorMsg(result).c_str());
            failedResults.push_back(GetValueResult{
                    .requestId = request.requestId,
                    .status = getErrorCode(result),
                    .prop = {},
            });
        } else {
            hardwareRequests.push_back(request);
        }
    }

    // The set of request Ids that we would send to hardware.
    //发送给硬件的一组请求 ID
    std::unordered_set<int64_t> hardwareRequestIds;
    for (const auto& request : hardwareRequests) {
        hardwareRequestIds.insert(request.requestId);
    }

    std::shared_ptr<GetValuesClient> client;
    {
        client = getOrCreateClient(&mGetValuesClients, callback, mPendingRequestPool);
    }

    // Register the pending hardware requests and also check for duplicate request Ids.
    //注册待处理的硬件请求并检查重复的请求 ID
    if (auto addRequestResult = client->addRequests(hardwareRequestIds); !addRequestResult.ok()) {
        ALOGE("getValues[%s]: failed to add pending requests, error: %s",
              toString(hardwareRequestIds).c_str(), getErrorMsg(addRequestResult).c_str());
        return toScopedAStatus(addRequestResult);
    }

    if (!failedResults.empty()) {
        // First send the failed results we already know back to the client.
        client->sendResults(std::move(failedResults));
    }

    if (hardwareRequests.empty()) {
        return ScopedAStatus::ok();
    }
	//调用到FakeVehicleHardware的getValues
    if (StatusCode status =
                mVehicleHardware->getValues(client->getResultCallback(), hardwareRequests);
        status != StatusCode::OK) {
        // If the hardware returns error, finish all the pending requests for this request because
        // we never expect hardware to call callback for these requests.
        //硬件返回错误，则完成此请求的所有待处理请求
        client->tryFinishRequests(hardwareRequestIds);
        ALOGE("getValues[%s]: failed to get value from VehicleHardware, status: %d",
              toString(hardwareRequestIds).c_str(), toInt(status));
        return ScopedAStatus::fromServiceSpecificErrorWithMessage(
                toInt(status), "failed to get value from VehicleHardware");
    }
    return ScopedAStatus::ok();
}
```

FakeVehicleHardware继承于IVehicleHardware，因此会调用FakeVehicleHardware的getValues

```C++
StatusCode FakeVehicleHardware::getValues(std::shared_ptr<const GetValuesCallback> callback,
                                          const std::vector<GetValueRequest>& requests) const {
    for (auto& request : requests) {
        if (FAKE_VEHICLEHARDWARE_DEBUG) {
            ALOGD("getValues(%d)", request.prop.prop);
        }

        // In a real VHAL implementation, you could either send the getValue request to vehicle bus
        // here in the binder thread, or you could send the request in getValue which runs in
        // the handler thread. If you decide to send the getValue request here, you should not
        // wait for the response here and the handler thread should handle the getValue response.
        mPendingGetValueRequests.addRequest(request, callback);
    }

    return StatusCode::OK;
}
```

把GetValue添加到RequestmPendingGetValueRequests

```C++
template <class CallbackType, class RequestType>
FakeVehicleHardware::PendingRequestHandler<CallbackType, RequestType>::PendingRequestHandler(
        FakeVehicleHardware* hardware)
    : mHardware(hardware) {
    // Don't initialize mThread in initialization list because mThread depends on mRequests and we
    // want mRequests to be initialized first.
    mThread = std::thread([this] {
        while (mRequests.waitForItems()) {
            handleRequestsOnce();
        }
    });
}

template <class CallbackType, class RequestType>
void FakeVehicleHardware::PendingRequestHandler<CallbackType, RequestType>::addRequest(
        RequestType request, std::shared_ptr<const CallbackType> callback) {
    //mRequests是ConcurrentQueue，会在上面PendingRequestHandler的线程中执行
    mRequests.push({
            request,
            callback,
    });
}
```

handleRequestsOnce方法

```C++
void FakeVehicleHardware::PendingRequestHandler<FakeVehicleHardware::GetValuesCallback,
                                                GetValueRequest>::handleRequestsOnce() {
    std::unordered_map<std::shared_ptr<const GetValuesCallback>, std::vector<GetValueResult>>
            callbackToResults;
    for (const auto& rwc : mRequests.flush()) {
        //FakeVehicleHardware的handleGetValueRequest
        auto result = mHardware->handleGetValueRequest(rwc.request);
        callbackToResults[rwc.callback].push_back(std::move(result));
    }
    for (const auto& [callback, results] : callbackToResults) {
        (*callback)(std::move(results));
    }
}

GetValueResult FakeVehicleHardware::handleGetValueRequest(const GetValueRequest& request) {
    auto result = getValue(request.prop);
    return getValueResult;
}

FakeVehicleHardware::ValueResultType FakeVehicleHardware::getValue(
       const VehiclePropValue& value) const {
    auto readResult = mServerSidePropStore->readValue(value);
    return std::move(readResult);
}
```

 mServerSidePropStore是VehiclePropertyStore。

FakeVehicleHardware模拟与实际硬件或车载总线交互的VHAL逻辑。它使用内存中的映射来存储属性值，并且不与实际的车载总线通信。

VehiclePropertyStore就是用来存储和管理车辆属性的配置。模拟器修改的配置也会通过这个保存，没有硬件相关的交互，就是存储数据，通知数据。

实际开发中FakeVehicleHardware需要修改调用驱动的接口，比如向CAN或者串口读写数据，或者使用socket通信，跟相关的ECU通信，可以接入SOME/IP或FDBus。

[(一) 车载以太网通信之SOME/IP协议_someip协议-CSDN博客](https://blog.csdn.net/zjfengdou30/article/details/125332151)  这篇SOME/IP系列文章很推荐。

[VHAL 接口  | Android Open Source Project](https://source.android.google.cn/docs/automotive/vhal/vhal-interface?hl=zh-cn)   车辆属性相关参考官方文档。
