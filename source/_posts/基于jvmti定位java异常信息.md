---
title: 基于jvmti定位java异常信息
date: 2019-05-19 23:03:45
tags:
- jvmti
categories: JVM实战
---
**背景描述**
JVMTI(JVM Tool Interface)位于jpda最底层，是Java虚拟机所提供的native编程接口。JVMTI可以提供性能分析、debug、内存管理、线程分析等功能。

JPDA 定义了一个完整独立的体系，它由三个相对独立的层次共同组成，而且规定了它们三者之间的交互方式，或者说定义了它们通信的接口。这三个层次由低到高分别是 Java 虚拟机工具接口（JVMTI），Java 调试线协议（JDWP）以及 Java 调试接口（JDI）。这三个模块把调试过程分解成几个很自然的概念：调试者（debugger）和被调试者（debuggee），以及他们中间的通信器。被调试者运行于我们想调试的 Java 虚拟机之上，它可以通过 JVMTI 这个标准接口，监控当前虚拟机的信息；调试者定义了用户可使用的调试接口，通过这些接口，用户可以对被调试虚拟机发送调试命令，同时调试者接受并显示调试结果。在调试者和被调试着之间，调试命令和调试结果，都是通过 JDWP 的通讯协议传输的。所有的命令被封装成 JDWP 命令包，通过传输层发送给被调试者，被调试者接收到 JDWP 命令包后，解析这个命令并转化为 JVMTI 的调用，在被调试者上运行。类似的，JVMTI 的运行结果，被格式化成 JDWP 数据包，发送给调试者并返回给 JDI 调用。而调试器开发人员就是通过 JDI 得到数据，发出指令。
<!-- more -->
![JDPA 模块层次.png](http://itstack.gitee.io/images_bed/img/other/%E5%9F%BA%E4%BA%8Ejvmti%E5%AE%9A%E4%BD%8Djava%E5%BC%82%E5%B8%B8%E4%BF%A1%E6%81%AF-01.png)

| 模块       | 层次          | 编程语言  |作用  |
| :-------: |:-------------:| :-----:| :-----:|
| JVMTI    | 底层 | C |获取及控制当前虚拟机状态 |
| JDWP    | 中介层     |   C |定义 JVMTI 和 JDI 交互的数据格式 |
| JDI | 高层     |    Java |提供 Java API 来远程控制被调试虚拟机 |


>Java 虚拟机工具接口（JVMTI）
JVMTI（Java Virtual Machine Tool Interface）即指 Java 虚拟机工具接口，它是一套由虚拟机直接提供的 native 接口，它处于整个 JPDA 体系的最底层，所有调试功能本质上都需要通过 JVMTI 来提供。通过这些接口，开发人员不仅调试在该虚拟机上运行的 Java 程序，还能查看它们运行的状态，设置回调函数，控制某些环境变量，从而优化程序性能。我们知道，JVMTI 的前身是 JVMDI 和 JVMPI，它们原来分别被用于提供调试 Java 程序以及 Java 程序调节性能的功能。在 J2SE 5.0 之后 JDK 取代了 JVMDI 和 JVMPI 这两套接口，JVMDI 在最新的 Java SE 6 中已经不提供支持，而 JVMPI 也计划在 Java SE 7 后被彻底取代。

>Java 调试线协议（JDWP）
JDWP（Java Debug Wire Protocol）是一个为 Java 调试而设计的一个通讯交互协议，它定义了调试器和被调试程序之间传递的信息的格式。在 JPDA 体系中，作为前端（front-end）的调试者（debugger）进程和后端（back-end）的被调试程序（debuggee）进程之间的交互数据的格式就是由 JDWP 来描述的，它详细完整地定义了请求命令、回应数据和错误代码，保证了前端和后端的 JVMTI 和 JDI 的通信通畅。比如在 Sun 公司提供的实现中，它提供了一个名为 jdwp.dll（jdwp.so）的动态链接库文件，这个动态库文件实现了一个 Agent，它会负责解析前端发出的请求或者命令，并将其转化为 JVMTI 调用，然后将 JVMTI 函数的返回值封装成 JDWP 数据发还给后端。

>Java 调试接口（JDI）
JDI（Java Debug Interface）是三个模块中最高层的接口，在多数的 JDK 中，它是由 Java 语言实现的。 JDI 由针对前端定义的接口组成，通过它，调试工具开发人员就能通过前端虚拟机上的调试器来远程操控后端虚拟机上被调试程序的运行，JDI 不仅能帮助开发人员格式化 JDWP 数据，而且还能为 JDWP 数据传输提供队列、缓存等优化服务。从理论上说，开发人员只需使用 JDWP 和 JVMTI 即可支持跨平台的远程调试，但是直接编写 JDWP 程序费时费力，而且效率不高。因此基于 Java 的 JDI 层的引入，简化了操作，提高了开发人员开发调试程序的效率。

**开发简述**
基于jvmti提供的接口服务，运用C++代码(win32-add_library)在Agent_OnLoad里开发监控服务，并生成dll文件。开发完成后在java代码中加入agentpath，这样就可以监控到我们需要的信息内容。

**环境准备**
1、Dev-C++
2、JetBrains CLion 2018.2.3
3、IntelliJ IDEA Community Edition 2018.3.1 x64
4、jdk1.8 64位
5、jvmti（在jdk安装目录下jdk1.8.0_45\include里，复制到和工程案例同层级目录下）

**配置信息**（路径相关修改为自己的）
1、C++开发工具Clion配置
1.1、配置位置；Settings->Build,Execution,Deployment->Toolchains
1.2、MinGM配置：D:\Program Files (x86)\Dev-Cpp\MinGW64
2、java调试时配置
2.1、配置位置：Run/Debug Configurations ->VM options
2.2、配置内容：-agentpath:E:\itstack\itstack.org\demo\jvmti\itstack-demo-jvmti-dll\cmake-build-debug\libitstack_demo_jvmti_dll.dll   

**代码示例**

c++ 代码块：
```java
#include <iostream>
#include <cstring>
#include "jvmti.h"

using namespace std;

//异常回调函数
static void JNICALL callbackException(jvmtiEnv *jvmti_env, JNIEnv *env, jthread thr, jmethodID methodId, jlocation location, jobject exception, jmethodID catch_method, jlocation catch_location) {

    // 获得方法对应的类
    jclass clazz;
    jvmti_env->GetMethodDeclaringClass(methodId, &clazz);

    // 获得类的签名
    char *class_signature;
    jvmti_env->GetClassSignature(clazz, &class_signature, nullptr);

    //过滤非本工程类信息
    string::size_type idx;
    string class_signature_str = class_signature;
    idx = class_signature_str.find("org/itstack");
    if (idx != 1) {
        return;
    }

    //异常类名称
    char *exception_class_name;
    jclass exception_class = env->GetObjectClass(exception);
    jvmti_env->GetClassSignature(exception_class, &exception_class_name, nullptr);

    // 获得方法名称
    char *method_name_ptr, *method_signature_ptr;
    jvmti_env->GetMethodName(methodId, &method_name_ptr, &method_signature_ptr, nullptr);

    //获取目标方法的起止地址和结束地址
    jlocation start_location_ptr;    //方法的起始位置
    jlocation end_location_ptr;      //用于方法的结束位置
    jvmti_env->GetMethodLocation(methodId, &start_location_ptr, &end_location_ptr);

    //输出测试结果
    cout << "测试结果-定位类的签名：" << class_signature << endl;
    cout << "测试结果-定位方法信息：" << method_name_ptr << " -> " << method_signature_ptr << endl;
    cout << "测试结果-定位方法位置：" << start_location_ptr << " -> " << end_location_ptr + 1 << endl;
    cout << "测试结果-异常类的名称：" << exception_class_name << endl;

    cout << "测试结果-输出异常信息(可以分析行号)：" << endl;
    jclass throwable_class = (*env).FindClass("java/lang/Throwable");
    jmethodID print_method = (*env).GetMethodID(throwable_class, "printStackTrace", "()V");
    (*env).CallVoidMethod(exception, print_method);

}

JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *vm, char *options, void *reserved) {
    jvmtiEnv *gb_jvmti = nullptr;
    //初始化
    vm->GetEnv(reinterpret_cast<void **>(&gb_jvmti), JVMTI_VERSION_1_0);
    // 创建一个新的环境
    jvmtiCapabilities caps;
    memset(&caps, 0, sizeof(caps));
    caps.can_signal_thread = 1;
    caps.can_get_owned_monitor_info = 1;
    caps.can_generate_method_entry_events = 1;
    caps.can_generate_exception_events = 1;
    caps.can_generate_vm_object_alloc_events = 1;
    caps.can_tag_objects = 1;
    // 设置当前环境
    gb_jvmti->AddCapabilities(&caps);
    // 创建一个新的回调函数
    jvmtiEventCallbacks callbacks;
    memset(&callbacks, 0, sizeof(callbacks));
    //异常回调
    callbacks.Exception = &callbackException;
    // 设置回调函数
    gb_jvmti->SetEventCallbacks(&callbacks, sizeof(callbacks));
    // 开启事件监听(JVMTI_EVENT_EXCEPTION)
    gb_jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_EXCEPTION, nullptr);
    return JNI_OK;
}

JNIEXPORT void JNICALL Agent_OnUnload(JavaVM *vm) {
}

```
java代码块：
```java
package org.itstack.demo.jvmti;
import java.util.logging.Logger;

public class TestLocationException {

    public static void main(String[] args) {
        Logger logger = Logger.getLogger("TestLocationException");
        try {
            User resource = new User();
            Object obj = resource.queryUserInfoById(null);
            logger.info("测试结果：" + obj);
        } catch (Exception e) {
            //屏蔽异常
        }
    }
}

class User {
    Logger logger = Logger.getLogger("User");
    public Object queryUserInfoById(String userId) {
        logger.info("根据用户Id获取用户信息" + userId);
        if (null == userId) {
            throw new NullPointerException("根据用户Id获取用户信息，空指针异常");
        }
        return userId;
    }
}
```
**测试结果**
```
四月 13, 2019 12:21:45 下午 org.itstack.demo.jvmti.User queryUserInfoById
信息: 根据用户Id获取用户信息null
测试结果-定位类的签名：Lorg/itstack/demo/jvmti/User;
测试结果-定位方法信息：queryUserInfoById -> (Ljava/lang/String;)Ljava/lang/Object;
测试结果-定位方法位置：0 -> 43
测试结果-异常类的名称：Ljava/lang/NullPointerException;
测试结果-输出异常信息(可以分析行号)：
java.lang.NullPointerException: 根据用户Id获取用户信息，空指针异常
	at org.itstack.demo.jvmti.User.queryUserInfoById(TestLocationException.java:23)
	at org.itstack.demo.jvmti.TestLocationException.main(TestLocationException.java:10)
```
**其他内容：**
1、[jvmti api](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html)
2、[JPDA 体系概览](https://www.ibm.com/developerworks/cn/java/j-lo-jpda1/index.html?ca=drs-)
3、*需要完整代码请留邮箱*
