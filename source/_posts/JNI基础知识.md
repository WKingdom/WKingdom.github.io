---
title: JNI基础知识
date: 2020-09-01 14:49:37
categories: 
- Android
- JNI
tag: 
- Android
- JNI
---
定义：Java Native Interface，即 Java本地接口 

作用： 使得Java与本地其他类型语言（如C、C++）交互 

JNI是Java调用 Native 语言的一种特性 JNI是属于Java 的，与 Android 无直接关系，实际中的驱动都是C/C++开发的,通过JNI,Java可以调用C/C++实现的驱动，从而扩展Java虚拟机的能力。

**JNI与NDK的关系**

JNI :Java平台JDK提供的一套非常强大的框架 Java Native Interface 相互调用交互通信：C C++ Native <-------> Java / Kotlin 

NDK：Android平台提供的Native开发工具集开发包 Native Development Kit 后面把开始的JNI，拿到NDK里面来并进行封装 （JNI，gcc，g++，...）



jni.h 有两份：  jni.h JDK版本与 NDK版本是不一样的： 

NDK版本：Sdk\ndk\21.0.6113669\toolchains\llvm\prebuilt\windows x86_64\sysroot\usr\include\jni.h 

JDK版本： Java\jdk1.8.0_131\include\jni.h

##### 签名规则

C/C++调用java类对象的属性/方法的签名规则

| java类型 | 属性类型符号 |
| ---- | ----- |
|boolean|Z|
|byte|B |
|char|C |
|short|S |
|int|I |
|long| J |
|float|F |
|double| D |
|void| V |
|object | L完整的类名；  String --> Llava/long/String; |
|array[数组的数据类型  | int[]  [I      double[][]   [[D |
|method(参数类型)返回值类型 | void name(int a,double b）  (ID)V |

##### C++ 与 Java 交互操作

NativeTest.java

```java
package com.xxx.test;
public class NativeTest{
	static {
 		System.loadLibrary("native-lib");
 	}
     public String name = "wtf"; // 等下 用native代码 修改为：nativeWtf
    
    // native系列函数：
	public native void changeName(); // 改变属性name
    
     // 被C调用的方法
	public int add(int number1,int number2){
 		return number1 + number2;
 	}
 }
```

 native-lib.cpp

```C++
#include <jni.h>
 #include <string>
 // 日志输出
#include <android/log.h>
 #define TAG "Wtf"
 // __VA_ARGS__ 代表...的可变参数
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, TAG,  __VA_ARGS__);
 #define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG,  __VA_ARGS__);
 #define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG,  __VA_ARGS__);
 extern "C"
 JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_changeName(JNIEnv* env,jobject obj) {
 	// 1.修改name
 	jclass testCls = env->GetObjectClass(obj);
 	jfieldID name_fid = env->GetFieldID(testCls, "name", "Ljava/lang/String;");
 	jstring value = env->NewStringUTF("nativeWtf");
 	env->SetObjectField(obj, name_fid, value);
 }

extern "C"
JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_callAddMathod(JNIEnv *env, jobject jobj) {
 	// 3. C++ 调用Java方法public int add(int number1,int number2){ return number1+number2;}
 	jclass testCls = env->GetObjectClass(jobj);
 	jmethodID addMid = env->GetMethodID(testCls, "add", "(II)I");
 	int result = env->CallIntMethod(jobj, addMid, 100000, 200000);
 	LOGE("result:%d\n", result);
 }
```

#####  JNI函数细节介绍

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_xxx_test_NativeTest_callAddMathod(JNIEnv *env, jobject jobj) {
    
extern "C" // 采用C的编译方式
JNIEXPORT  // JNI重要标记关键字，标记为该方法可以被外部调用
jstring    // 代表 java 中的 String 
JNICALL    // 也是一个关键字，（可以少的）jni call (约束函数入栈顺序，和堆栈内存清理的规则)
jobject    // java传递下来的对象，就是NativeTest对象
jclass     // java传递下来的class对象，就是NativeTest class
```

extern "C" 的原因

```C++
C++的情况如下：
xxxxx(JNIEnv * env, ...) {
 env->函数();
 }

C的情况如下：
xxxxx(JNIEnv * env, ...) {
 (*env)->函数();
 }
```

原因是： C:\Program Files\Java\jdk1.8.0_131\include\jni.h里面中，搜索查询，struct JNIEnv

```C++
struct JNIEnv_;
#ifdef __cplusplus // 如果是C++ 下面就直接使用JNIEnv结构体  【xxx(JNIEnv * env) == 一级指针】
typedef JNIEnv_ JNIEnv;
#else // 如果是C 下面就直接使用 JNINativeInterface_ 结构体 指针(一级指针)  【xxx(JNIEnv * env) == 二级指针】 
typedef const struct JNINativeInterface_ *JNIEnv;
#endif
```

##### JNI数组操作

```java
public native void testArrayAction(int count, String textInfo, int[] ints, String[] strs); 
```

```C++
 // jint == int
 // jstring == String
 // jintArray == int[]
 // jobjectArray == 引用类型的对象，例如:String[]
 extern "C"
 JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_testArrayAction(JNIEnv *env,
 	jobject instance, jint count, jstring text_info,
	jintArray ints, jobjectArray strs) {
 	jint _count = count;
 	LOGD("_count:%d\n", _count)
    // 0:代表就在本地空间操作，内部不需要建立Copy机制（一般情况下不会建立Copy机制）
 	const char * _text_info = env->GetStringUTFChars(text_info, NULL); 
	LOGE("_text_info:%s\n", _text_info)
    // 释放工作，必须要做
 	env->ReleaseStringUTFChars(text_info, _text_info); 
     
    jint * _ints = env->GetIntArrayElements(ints, NULL);
	int intsLen = env->GetArrayLength(ints);
 	for (int i = 0; i < intsLen; ++i) {
		*(_ints+i) = (i + 1000001);
 		LOGD("C++ _ints item:%d\n", *(_ints+i))
 		// JNI_OK 0 == 代表 先用操纵杆刷新到JVM 再释放C++层数组
		// JNI_COMMIT 1 == 代表 用操纵杆刷新到JVM
 		// JNI_ABORT 2 == 代表 释放C++层数组
		env->ReleaseIntArrayElements(ints, _ints, JNI_OK);
 	}
 
	int strsLen = env->GetArrayLength(strs);
 	for (int i = 0; i < strsLen; ++i) {
 		jobject item = env->GetObjectArrayElement(strs, i);
 		jstring itemStr = (jstring) item;
 		const char * itemStr_ = env->GetStringUTFChars(itemStr, NULL);
 		LOGI("C++ 修改前itemStr_:%s\n", itemStr_);
 		env->ReleaseStringUTFChars(itemStr, itemStr_);
 		jstring value = env->NewStringUTF("AAAAAAA");
 		env->SetObjectArrayElement(strs, i, value);
 		jobject item2 = env->GetObjectArrayElement(strs, i);
 		jstring itemStr2 = (jstring) item2;
		 const char * itemStr_2 = env->GetStringUTFChars(itemStr2, NULL);
 		LOGI("C++ 修改后itemStr_2:%s\n", itemStr_2);
 		env->ReleaseStringUTFChars(itemStr2, itemStr_2);
 	}
```

##### JNI对象操作

```java
public native void putObject(Student student, String str); // 传递引用类型，传递对象

//Student.java
public class Student {
    public String name;
    public String getName() {
 		return name;
 	}
 	public void setName(String name) {
 		Log.d(TAG, "Java setName name:" + name);
 	this.name = name;
 	}
}
```

```C++
extern "C"
 JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_putObject(JNIEnv *env,
 	jobject instance,
 	jobject student,
 	jstring str) {
	const char * _str = env->GetStringUTFChars(str, NULL);
 	LOGD("_str:%s\n", _str)
 	env->ReleaseStringUTFChars(str, _str);
 	jclass mStudentClass = env->GetObjectClass(student);
 	// toString
 	jmethodID toStringMethod = env->GetMethodID(mStudentClass, "toString", "()Ljava/lang/String;");
 	jstring results = (jstring) env->CallObjectMethod(student, toStringMethod);
 	const char * result = env->GetStringUTFChars(results, NULL);
 	LOGD("C++ toString:%s\n", result)
 	env->ReleaseStringUTFChars(results, result);
 	// setName
 	jmethodID setNameMethod = env->GetMethodID(mStudentClass, "setName", "(Ljava/lang/String;)V");
 	jstring nameS = env->NewStringUTF("Derry");
 	env->CallVoidMethod(student, setNameMethod, nameS);
 	// getName
 	jmethodID getNameMethod = env->GetMethodID(mStudentClass, "getName", "()Ljava/lang/String;");
 	jstring  nameS2 = (jstring) env->CallObjectMethod(student, getNameMethod);
 	const char * nameS3 = env->GetStringUTFChars(nameS2, NULL);
 	LOGD("C++ getName:%s\n", nameS3)
 	env->ReleaseStringUTFChars(nameS2, nameS3);
 
 	env->DeleteLocalRef(mStudentClass);
}

```

##### JNI全局引用与局部引用

```C++
// 在JNI函数中，会有，局部引用，全局引用
// 默认情况下，都是局部引用，局部引用 在JNI函数结束执行后，会自动回收所有的局部引用
// 全局引用，需要开发者手动去提升为全局引用，全局引用必须 手动释放，否则不会被回收，所以通常情况下，会在Activity的onDestroy中 释放全局引用
jclass mStudentClass  = nullptr;
extern "C"
JNIEXPORT void JNICALL
Java_com_xxx_test_NativeTest_putObject(JNIEnv *env, jobject thiz){
    {
 	if (!mStudentClass) {
 	// 提升为全局引用，让JNI函数结束后，不要自动去回收 局部引用
		jclass tempStuClass = env->FindClass("com/xxx/test/Student");;
 		mStudentClass = (jclass) env->NewGlobalRef((jobject)tempStuClass); // 提升为全局引用
		env->DeleteLocalRef(tempDogClass);
 	}
}
extern "C"
JNIEXPORT void JNICALL
Java_com_xxx_test_NativeTest_release(JNIEnv *env, jobject thiz) {
 if (mStudentClass) {
 	env->DeleteGlobalRef(mStudentClass); // 手动释放全局引用成员dogClass
 	mStudentClass = nullptr;
 	LOGI("手动释放全局引用成功success...")
 	}
 }
```

##### 动态注册

Java_com_xxx_test_NativeTest_putObject这样的方式是静态注册，还有动态注册方法。

动态的优点：1.被反编译后安全性高一点， 2.在native中的调用，函数名简洁， 3. 编译后的函数标记较短一些

```java
//native方法
public native void dynamicJavaMethod01(); // 动态注册1
public native int dynamicJavaMethod02(String valueStr); // 动态注册2
```

 动态注册代码

```C++
JavaVM *jVm = nullptr;
const char *testClassName = "com/xxx/test/NativeTest";
// native 真正的函数
// void dynamicMethod01(JNIEnv *env, jobject thiz) { 
void dynamicMethod01() { // 如果用不到JNIEnv jobject ，可以不用写
    LOGD("dynamicMethod01...");
 }
 int dynamicMethod02(JNIEnv *env, jobject thiz, jstring valueStr) { 
    const char *text = env->GetStringUTFChars(valueStr, nullptr);
    LOGD("dynamicMethod02... %s", text);
    env->ReleaseStringUTFChars(valueStr, text);
    return 200;
 }
 /*
     typedef struct {
        const char* name;       // 函数名
        const char* signature; // 函数的签名
        void*       fnPtr;     // 函数指针
     } JNINativeMethod;
     */
 static const JNINativeMethod jniNativeMethod[] = {
        {"dynamicJavaMethod01", "()V",                   (void *) (dynamicMethod01)
        {"dynamicJavaMethod02", "(Ljava/lang/String;)I", (int *) (dynamicMethod02)
 };

// JNI JNI_OnLoad函数
 extern "C"
 JNIEXPORT jint JNI_OnLoad(JavaVM *javaVm, void *) {
    ::jVm = javaVm;
    // 动态注册
    JNIEnv *jniEnv = nullptr;
    int result = javaVm->GetEnv(reinterpret_cast<void **>(&jniEnv), JNI_VERSION_1_6);
    // result 等于0  就是成功
    if (result != JNI_OK) {
        return -1; // 会奔溃，故意奔溃
    }
    LOGE("System.loadLibrary ---》 JNI Load init");
    jclass testClass = jniEnv->FindClass(testClassName);
    // jint RegisterNatives(Class, 数组==jniNativeMethod， 注册的数量 = 2)
    jniEnv->RegisterNatives(testClass,
                            jniNativeMethod,
                            sizeof(jniNativeMethod) / sizeof(JNINativeMethod)
    LOGE("动态注册");
    return JNI_VERSION_1_6;
 }
```

##### JNI线程

```java
public native void naitveThread(); // Java层 调用 Native层 的函数，完成JNI线程
public native void closeThread(); // 释放全局引用
```

```C++
#include <pthread.h>

class MyContext {
 public:
    JNIEnv *jniEnv = nullptr;  // 不能跨线程 ，会奔溃
    jobject instance = nullptr; // 不能跨线程 ，会奔溃
};
void *myThreadTaskAction(void *pVoid) { // 当前是异步线程
    LOGE("myThreadTaskAction run");
    // 这两个是必须要的
    // JNIEnv *env
    // jobject thiz   OK
    MyContext * myContext = static_cast<MyContext *>(pVoid);
    // TODO 解决方式 （安卓进程只有一个 JavaVM，是全局的，是可以跨越线程的）
    JNIEnv * jniEnv = nullptr; // 全新的JNIEnv  异步线程里面操作
    jint attachResult = ::jVm->AttachCurrentThread(&jniEnv, nullptr); 
    if (attachResult != JNI_OK) {
        return 0; // 附加失败，返回了
    }
    // 1.拿到class
    jclass testClass = jniEnv->GetObjectClass(myContext->instance);
    // 2.拿到方法
    jmethodID updateUI = jniEnv->GetMethodID(testClass, "updateUI", "()V");
    // 3.调用
    jniEnv->CallVoidMethod(myContext->instance, updateUI);
    ::jVm->DetachCurrentThread(); // 必须解除附加，否则报错
    LOGE("C++ 异步线程OK")
    return nullptr;
 }
                                                     
 extern "C"
 JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_naitveThread(JNIEnv *env, jobject job){
    MyContext * myContext = new MyContext;
    myContext->jniEnv = env;
    // myContext->instance = job; // 默认是局部引用，会奔溃
    myContext->instance = env->NewGlobalRef(job); // 提升全局引用
    pthread_t pid;
    pthread_create(&pid, nullptr, myThreadTaskAction, myContext);
    pthread_join(pid, nullptr);
 }
 extern "C"
 JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_closeThread(JNIEnv *env, jobject job){
    // 做释放工作
}
// 1. JavaVM全局，绑定当前进程， 只有一个地址
// 2. JNIEnv线程绑定， 绑定主线程，绑定子线程
// 3. jobject 谁调用JNI函数，谁的实例会给jobject
// JNIEnv *env 不能跨越线程，否则崩溃，  他可以跨越函数 【
// jobject thiz 不能跨越线程，否则崩溃，不能跨越函数，否则崩溃
// JavaVM 能够跨越线程，能够跨越函数
```

##### JNI数组排序

NativeTestKt.kt

```kotlin
class NativeTestKt{
// static { System.loadLibrary("native-lib"); }
companion object { // 派生类里面全部都是 相当于Java的static成员
	init {
 		System.loadLibrary("native-lib")
 	}
 }
 // public native void sort(int[] arr);
 external fun sort(arr: IntArray) // 数组排序
}

```

 native-lib.cpp：

```C++
/ TODO 01.数组排序
// 比较的函数
int compare(const jint *number1, const jint *number2){
 	return *number1 - *number2;
 }
 extern "C"
 JNIEXPORT void JNICALL
 Java_com_xxx_test_NativeTest_sort(JNIEnv *env, jobject thiz, jintArray arr) {
	// 对arr 进行排序 （sort）
	jint* intArray = env->GetIntArrayElements(arr, nullptr);
 	int length = env->GetArrayLength(arr);
 	//NDK 很大的工具链（Java JNI，C++，stdlib ....） 工具箱
	// 第一个参数：void* 数组的首地址
	// 第二个参数：数组的大小长度
	// 第三个参数：数组元素数据类型的大小
	// 第四个参数：数组的一个比较方法指针（Comparable）
	qsort(intArray, length, sizeof(int),
 	reinterpret_cast<int (*)(const void *, const void *)>(compare));
 	// 同步数组的数据给java 数组intArray 并不是arr ，可以简单的理解为copy
 	// 0 : 既要同步数据给arr ,又要释放intArray，会排序
	// JNI_COMMIT: 会同步数据给arr ，但是不会释放intArray，会排序
	// JNI_ABORT: 不同步数据给arr ，但是会释放intArray，所以上层看到就并不会排序
	env->ReleaseIntArrayElements(arr, intArray, JNI_COMMIT);
 }
```

##### 异常捕获

```java
 // 这里定义变量
static String name1 = "T1";
public static native void exception();
public static native void exception2() throws NoSuchFieldException;
public static native void exception3();
```

```C++
// 异常方式一： 【C++处理时异常】
extern "C"
JNIEXPORT void JNICALL
Java_com_xxx_test_NativeTest_exception(JNIEnv *env, jclass clazz) {
    // 想操作name1，但是写成了name111,没有name111就会在native层崩溃
    jfieldID f_id = env->GetStaticFieldID(clazz, "name111", "Ljava/lang/String;");
    // name111 拿不到报错的话，就拿 name1
 	jthrowable throwable = env->ExceptionOccurred(); // 检测本次函数执行，到底有没有异
 	if (throwable){
 		//先把异常清除
		LOGD("native层：检测到 有异常...");
 		// 清除异常
		env->ExceptionClear();
 		// 重新获取 name1 属性
		f_id = env->GetStaticFieldID(clazz, "name1", "Ljava/lang/String;");
    }
}
// 异常方式二：【C++处理时异常】 往Java层抛出异常
extern "C"
JNIEXPORT void JNICALL
Java_com_xxx_test_NativeTest_exception2(JNIEnv *env, jclass clazz) {
 	// 想操作name1，但是写成了name111,没有name111就会在native层崩溃
	jfieldID f_id = env->GetStaticFieldID(clazz, "name111", "Ljava/lang/String;");
	//name111拿不到报错的话，给java层抛一个异常
	jthrowable throwable = env->ExceptionOccurred();  // 检测本次函数执行，到底有没有异常
	// 给java层抛一个异常
	if (throwable){
 		// 清除异常
		env->ExceptionClear();
 		// Throw抛一个java的Throwable 对象
		jclass no_such_clz = env->FindClass("java/lang/NoSuchFieldException");
 		env->ThrowNew(no_such_clz,"NoSuchFieldException name111!");
	 }
 }
// 异常方式三：Java的方法抛出了异常，然后native去清除
// 注意：Java的异常native层无法捕获
extern "C"
JNIEXPORT void JNICALL
Java_com_xxx_test_NativeTest_exception3(JNIEnv *env, jclass clazz) {
 	//show方法中抛出异常
 	jmethodID showMID = env->GetStaticMethodID(clazz, "show", "()V");
 	env->CallStaticVoidMethod(clazz, showMID);
 	//上面的这句话：env->CallStaticVoidMethod(clazz, showMID)会崩溃
	LOGI("native层：>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>");
 	// 不是马上就奔溃
	if (env->ExceptionCheck()) {
 		env->ExceptionDescribe();// 输出描述
		env->ExceptionClear();// 清除异常
	}
}

```

[JNI接口文档](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)

[NDK JNI提示](https://developer.android.google.cn/training/articles/perf-jni?hl=zh-cn)