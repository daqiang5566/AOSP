## 前言
上文介绍完JNI的一些基本操作后，我们接下来讲zygote进程的下半段

之前《Android系统启动流程之zygote进程(一)》中讲了zygote进程的触发、参数的解析过程以及虚拟机的创建，虚拟机创建好了，
我们就正式进入Java环境了，也算是到了我们比较熟悉的领域

本文主要讲解以下内容

- 性能统计

- 资源预加载


本文涉及到的文件
```
platform/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
platform/frameworks/base/core/java/com/android/internal/logging/MetricsLogger.java
platform/frameworks/base/core/java/com/android/internal/logging/EventLogTags.logtags
platform/system/core/logcat/event.logtags
platform/build/tools/java-event-log-tags.py
platform/frameworks/base/core/jni/android_util_EventLog.cpp
platform/frameworks/base/core/java/android/os/Trace.java
platform/frameworks/base/core/jni/android_os_Trace.cpp
platform/system/core/libcutils/trace-dev.c
```

## 一、性能统计

定义在platform/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java


```java
public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation(); //设置标记，不允许新建线程

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0); //设置zygote进程组id为zygote的pid
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        try {
            // Report Zygote start time to tron unless it is a runtime restart
            if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
                MetricsLogger.histogram(null, "boot_zygote_init",
                        (int) SystemClock.elapsedRealtime());//记录boot_zygote_init时间戳
            }

            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
            BootTimingsTraceLog bootTimingsTraceLog = new BootTimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            bootTimingsTraceLog.traceBegin("ZygoteInit"); //跟踪调试ZygoteInit
            RuntimeInit.enableDdms(); //注册DDms的处理类
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start(); //开始性能统计
            ...
            SamplingProfilerIntegration.writeZygoteSnapshot();//结束性能统计并写入文件

}

```

main函数最开始new了一个ZygoteServer，这个后续会用到，然后设置标记，不允许新建线程，为什么不允许多线程呢？
这主要是担心用户创建app时，多线程情况下某些预先加载的资源没加载好，这时去调用会出问题. 接着设置了zygote进程的进程组id，
最后便是一系列性能统计相关的动作

### 1.1 事件日志记录

事件日志记录是Android系统中是比较常见的，主要作用就是打印日志，只是做了一些包装，还动用了脚本，所以有必要深入讲解下，
我们就以histogram为例


#### 1.1.1 histogram

定义在platform/frameworks/base/core/java/com/android/internal/logging/MetricsLogger.java

```java
    /** Increment the bucket with the integer label on the histogram with the given name. */
    public void histogram(String name, int bucket) {
        // see LogHistogram in system/core/libmetricslogger/metrics_logger.cpp
        EventLogTags.writeSysuiHistogram(name, bucket);
        saveLog(new LogMaker(MetricsEvent.RESERVED_FOR_LOGBUILDER_HISTOGRAM)
                        .setCounterName(name)
                        .setCounterBucket(bucket)
                        .setCounterValue(1)
                        .serialize());
    }
```

EventLogTags.writeSysuiHistogram 是找不到源码的，因为这个EventLogTags压根就不存在，因为它被写成了一种脚本语言，
就像之前的Android Init Language,脚本都是以.logtags 结尾，比如这个EventLogTags.writeSysuiHistogram对应的脚本定义在
platform/frameworks/base/core/java/com/android/internal/logging/EventLogTags.logtags


#### 1.1.2 EventLogTags.logtags

```text
# See system/core/logcat/event.logtags for a description of the format of this file.

option java_package com.android.internal.logging;

# interaction logs
524287 sysui_view_visibility (category|1|5),(visible|1|6)
524288 sysui_action (category|1|5),(pkg|3)
524292 sysui_multi_action (content|4)
524290 sysui_count (name|3),(increment|1)
524291 sysui_histogram (name|3),(bucket|1)

```

他说对于这个脚本的描述在system/core/logcat/event.logtags
我们找下platform/system/core/logcat/event.logtags


```text
# The entries in this file map a sparse set of log tag numbers to tag names.
# This is installed on the device, in /system/etc, and parsed by logcat.
#
# Tag numbers are decimal integers, from 0 to 2^31.  (Let's leave the
# negative values alone for now.)
#
# Tag names are one or more ASCII letters and numbers or underscores, i.e.
# "[A-Z][a-z][0-9]_".  Do not include spaces or punctuation (the former
# impacts log readability, the latter makes regex searches more annoying).
#
# Tag numbers and names are separated by whitespace.  Blank lines and lines
# starting with '#' are ignored.
#
# Optionally, after the tag names can be put a description for the value(s)
# of the tag. Description are in the format
#    (<name>|data type[|data unit])
# Multiple values are separated by commas.
#
# The data type is a number from the following values:
# 1: int
# 2: long
# 3: string
# 4: list
# 5: float
#
# The data unit is a number taken from the following list:
# 1: Number of objects
# 2: Number of bytes
# 3: Number of milliseconds
# 4: Number of allocations
# 5: Id
# 6: Percent
# Default value for data of type int/long is 2 (bytes).
#
# TODO: generate ".java" and ".h" files with integer constants from this file.
```

从描述中来看，这个文件将一些数字和tag名字一一对应起来，数字呢取值是0~2^31，tag名字只能用字母数字下划线（不能包含空格和标点符号），
tag名字后面跟一堆描述，格式是
```text
(<name>|data type[|data unit])
```
其实说白了，tag名字就相当于函数名字，这些描述相当于参数，name是参数名，data type是参数类型，data unit是参数类型的描述

data type用数字表示，有5种
```text
# 1: int
# 2: long
# 3: string
# 4: list
# 5: float
```
data unit也用数字表示，有6种, 默认是2
```text
# The data unit is a number taken from the following list:
# 1: Number of objects
# 2: Number of bytes
# 3: Number of milliseconds
# 4: Number of allocations
# 5: Id
# 6: Percent
# Default value for data of type int/long is 2 (bytes).
```


我们以之前的EventLogTags.writeSysuiHistogram为例子
```text
524291 sysui_histogram (name|3),(bucket|1)
```

524291是数字，这个数字会对应方法writeSysuiHistogram，第一个参数是String name,第二个参数是int bucket,也就是

```java
writeSysuiHistogram(String name, int bucket)
```

那EventLogTags.writeSysuiHistogram这个的实现在哪儿呢，我们只是找到了脚本，就像init.rc一样，这些脚本总是有解析它的地方啊，
这个其实是在编译过程中实现的，有一个python文件platform/build/tools/java-event-log-tags.py,会在编译时进行转换，
将xxx.logtags文件转为xxx.java,具体就是把之前的数字用静态变量存起来，tag名字和描述就生成方法，方法体中用EventLog代理，实现如下：

#### 1.1.3 java-event-log-tags.py

```python

for t in tagfile.tags:
  if t.description:
    buffer.write("\n  /** %d %s %s */\n" % (t.tagnum, t.tagname, t.description))
  else:
    buffer.write("\n  /** %d %s */\n" % (t.tagnum, t.tagname))

  buffer.write("  public static final int %s = %d;\n" %
               (t.tagname.upper(), t.tagnum))

def javaName(name):
  out = name[0].lower() + re.sub(r"[^A-Za-z0-9]", "", name.title())[1:]
  if out in keywords:
    out += "_"
  return out

javaTypes = ["ERROR", "int", "long", "String", "Object[]", "float"]
for t in tagfile.tags:
  methodName = javaName("write_" + t.tagname)
  if t.description:
    args = [arg.strip("() ").split("|") for arg in t.description.split(",")]
  else:
    args = []
  argTypesNames = ", ".join([javaTypes[int(arg[1])] + " " + javaName(arg[0]) for arg in args])
  argNames = "".join([", " + javaName(arg[0]) for arg in args])
  buffer.write("\n  public static void %s(%s) {" % (methodName, argTypesNames))
  buffer.write("\n    android.util.EventLog.writeEvent(%s%s);" % (t.tagname.upper(), argNames))
  buffer.write("\n  }\n")

buffer.write("}\n");
```

比如之前EventLogTags.logtags经过java-event-log-tags.py处理后就大概长这个样

```java
package com.android.internal.logging;

public class EventLogTags{
    
    private EventLogTags(){}
    
    ... //省略一些常量

    public static final int SYSUI_HISTOGRAM=524291;
    
    ... //省略一些方法

    public static void writeSysuiHistogram(String name,int bucket){
        android.util.EventLog.writeEvent(SYSUI_HISTOGRAM,name,bucket);
    }
    
}
```

所以EventLogTags.writeSysuiHistogram最终是调用了EventLog.writeEvent函数，这个函数是一个native函数，它的实现在
platform/frameworks/base/core/jni/android_util_EventLog.cpp

#### 1.1.3 android_util_EventLog

```C++
/*
 * JNI registration.
 */
static const JNINativeMethod gRegisterMethods[] = {
    /* name, signature, funcPtr */
    { "writeEvent", "(II)I", (void*) android_util_EventLog_writeEvent_Integer },
    { "writeEvent", "(IJ)I", (void*) android_util_EventLog_writeEvent_Long },
    { "writeEvent", "(IF)I", (void*) android_util_EventLog_writeEvent_Float },
    { "writeEvent",
      "(ILjava/lang/String;)I",
      (void*) android_util_EventLog_writeEvent_String
    },
    { "writeEvent",
      "(I[Ljava/lang/Object;)I",
      (void*) android_util_EventLog_writeEvent_Array
    },
```

我们看到writeEvent(int tag, Object... list)对应的函数是android_util_EventLog_writeEvent_Array

```C++
static jint android_util_EventLog_writeEvent_Array(JNIEnv* env, jobject clazz,
                                                   jint tag, jobjectArray value) {
    android_log_event_list ctx(tag);

    if (value == NULL) {
        ctx << "[NULL]";
        return ctx.write();
    }

    jsize copied = 0, num = env->GetArrayLength(value);
    for (; copied < num && copied < 255; ++copied) {
        if (ctx.status()) break;
        jobject item = env->GetObjectArrayElement(value, copied);
        if (item == NULL) {
            ctx << "NULL";
        } else if (env->IsInstanceOf(item, gStringClass)) {
            const char *str = env->GetStringUTFChars((jstring) item, NULL);
            ctx << str;
            env->ReleaseStringUTFChars((jstring) item, str);
        } else if (env->IsInstanceOf(item, gIntegerClass)) {
            ctx << (int32_t)env->GetIntField(item, gIntegerValueID);
        } else if (env->IsInstanceOf(item, gLongClass)) {
            ctx << (int64_t)env->GetLongField(item, gLongValueID);
        } else if (env->IsInstanceOf(item, gFloatClass)) {
            ctx << (float)env->GetFloatField(item, gFloatValueID);
        } else {
            jniThrowException(env,
                    "java/lang/IllegalArgumentException",
                    "Invalid payload item type");
            return -1;
        }
        env->DeleteLocalRef(item);
    }
    return ctx.write();
}
```

writeEvent有许多函数，但是基本都是向ctx里面拼接字符，然后调用ctx的write函数

```C++
  int write(log_id_t id = LOG_ID_EVENTS) {
    int retval = android_log_write_list(ctx, id);
    if (retval < 0) ret = retval;
    return ret;
  }
```

android_log_write_list里面写得比较复杂，因为这涉及到日志系统的处理流程，后续再专门拿一篇文章来讲，
android_log_write_list函数并不会直接输出日志，只是把要输出的日志内容加入到一个链表中，而日志系统会循环这个链表，最终输出

到这里，我们就理清了事件日志记录的全过程，主要是有一个脚本转化的过程，以后所有以.logtags结尾的文件都可以这样理解了


#### 1.2.1 traceBegin

定义在platform/frameworks/base/core/java/android/os/Trace.java

```java



    public static void traceBegin(long traceTag, String methodName) {
        if (isTagEnabled(traceTag)) {
            nativeTraceBegin(traceTag, methodName);
        }
    }
    
    public static boolean isTagEnabled(long traceTag) {
        long tags = sEnabledTags;
        if (tags == TRACE_TAG_NOT_READY) {
            tags = cacheEnabledTags();
        }
        return (tags & traceTag) != 0;
    }
     
    @FastNative
    private static native void nativeTraceBegin(long tag, String name);

```

traceBegin先判断了下一个状态值sEnabledTags，如果满足条件就调用nativeTraceBegin

#### 1.2.2 nativeTraceBegin

定义在platform/frameworks/base/core/jni/android_os_Trace.cpp

```C++
static const JNINativeMethod gTraceMethods[] = {

    // ----------- @FastNative  ----------------
    { "nativeTraceBegin",
            "(JLjava/lang/String;)V",
            (void*)android_os_Trace_nativeTraceBegin },
    { "nativeTraceEnd",
            "(J)V",
            (void*)android_os_Trace_nativeTraceEnd },
};
```

这里提一下@FastNative这个注解，这是ART 8.0增加的，可以提升JNI调用速度，参考https://blog.csdn.net/zhangbijun1230/article/details/80562747
nativeTraceBegin对应的函数是android_os_Trace_nativeTraceBegin

```C++
static void android_os_Trace_nativeTraceBegin(JNIEnv* env, jclass clazz,
        jlong tag, jstring nameStr) {
    ScopedStringChars jchars(env, nameStr); //对String的封装
    String8 utf8Chars(reinterpret_cast<const char16_t*>(jchars.get()), jchars.size());//也是对String的封装
    sanitizeString(utf8Chars);//将 /0, 回车，| 转为空格

    ALOGV("%s: %" PRId64 " %s", __FUNCTION__, tag, utf8Chars.string());
    atrace_begin(tag, utf8Chars.string());
}
```

这里是将传进来和参数转为C里面封装的String，然后将 /0, 回车，| 转为空格，输出一段日志后就调用atrace_begin
这里稍微讲一下C中如何寻找一些函数的定义，比如这里的atrace_begin函数并不在该源文件中，怎样快速找到对应的实现

首先Souce Insight导入文件时，尽量把一个文件夹一起导入，因为相关的源码都基本放在同一个文件夹下，这样就可以迅速找到
但是有时源码之间相差很远，这时我们就要观察include，看哪个比较像

```C++
#include <inttypes.h>

#include <cutils/trace.h>
#include <utils/String8.h>
#include <log/log.h>

#include <JNIHelp.h>
#include <ScopedUtfChars.h>
#include <ScopedStringChars.h>
```
该源文件中就这几个头文件，一看就知道应该是cutils/trace.h，所以就去对应目录platform/system/core/xxx找，基本就可以找到
另外再推荐一个Windows的搜索工具Everything，比如你找不到trace.h,直接搜索就能找到

```C
static inline void atrace_begin(uint64_t tag, const char* name)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_begin_body(const char*);
        atrace_begin_body(name);
    }
}
```

一般来讲头文件只做函数声明，不做代码实现，这里似乎实现了代码，其实并没有，这个函数是inline函数，本身会被消除掉，
也就是说在编译后，会把调用这个函数的地方替换为方法体中的代码，CC_UNLIKELY主要是做代码优化的，本身没有什么逻辑，
我们主要看下atrace_begin_body，这里就涉及到寻找.h的实现类

头文件的实现类xx.c，xx.cpp基本也就在对应的目录下，名字也不会差太远，
另外可以观察目录下的Android.mk或者Android.bp这些编译文件，里面会写包含了哪些源文件
```text
        android: {
            srcs: libcutils_nonwindows_sources + [
                "android_reboot.c",
                "ashmem-dev.c",
                "klog.cpp",
                "partition_utils.c",
                "properties.cpp",
                "qtaguid.c",
                "trace-dev.c",
                "uevent.c",
            ],
            sanitize: {
                misc_undefined: ["integer"],
            },
        },
```

trace.h的实现类叫trace-dev.c,定义在platform/system/core/libcutils/trace-dev.c


#### 1.2.3 atrace_begin_body

定义在platform/system/core/libcutils/trace-dev.c

```C++
void atrace_begin_body(const char* name)
{
    char buf[ATRACE_MESSAGE_LENGTH];

    int len = snprintf(buf, sizeof(buf), "B|%d|%s", getpid(), name); //拼接pid
    if (len >= (int) sizeof(buf)) {
        ALOGW("Truncated name in %s: %s\n", __FUNCTION__, name);
        len = sizeof(buf) - 1;
    }
    write(atrace_marker_fd, buf, len);//写入trace_marker文件
}


atrace_marker_fd = open("/sys/kernel/debug/tracing/trace_marker", O_WRONLY | O_CLOEXEC);

```

这就是最终干活的地方了，将pid拼接一下，将拼接的结果写入文件/sys/kernel/debug/tracing/trace_marker

### 1.3 enableDdms

```java

    static final void enableDdms() {
        // Register handlers for DDM messages.
        android.ddm.DdmRegister.registerHandlers();
    }

    public static void registerHandlers() {
        if (false)
            Log.v("ddm", "Registering DDM message handlers");
        DdmHandleHello.register();
        DdmHandleThread.register();
        DdmHandleHeap.register();
        DdmHandleNativeHeap.register();
        DdmHandleProfiling.register();
        DdmHandleExit.register();
        DdmHandleViewDebug.register();

        DdmServer.registrationComplete();
    }
    
    public static void register() {
        DdmServer.registerHandler(CHUNK_THEN, mInstance);
        DdmServer.registerHandler(CHUNK_THST, mInstance);
        DdmServer.registerHandler(CHUNK_STKL, mInstance);
    }
    
    public static void registerHandler(int type, ChunkHandler handler) {
        if (handler == null) {
            throw new NullPointerException("handler == null");
        }
        synchronized (mHandlerMap) {
            if (mHandlerMap.get(type) != null)
                throw new RuntimeException("type " + Integer.toHexString(type)
                    + " already registered");

            mHandlerMap.put(type, handler);
        }
    }    
```

这个比较简单，就是重复调用registerHandler函数，将一个int值（key）和一个处理类（value）放入HashMap一一映射起来

#### 1.1 参数解析

```java
public static void main(String argv[]) {

            ...

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
            zygoteServer.registerServerSocket(socketName);
            ...
}
```

```java
 public static void main(String argv[]) {

            ...

            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            } else {
                Zygote.resetNicePriority();
            }

            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
            gcAndFinalize();
            bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

            bootTimingsTraceLog.traceEnd(); // ZygoteInit
            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            ...

}
```


```java
 public static void main(String argv[]) {

            ...

            // Zygote process unmounts root storage spaces.
            Zygote.nativeUnmountStorageOnInit();

            // Set seccomp policy
            Seccomp.setPolicy();

            ZygoteHooks.stopZygoteNoThreadCreation();

            if (startSystemServer) {
                startSystemServer(abiList, socketName, zygoteServer);
            }

            Log.i(TAG, "Accepting command socket connections");
            zygoteServer.runSelectLoop(abiList);

            zygoteServer.closeServerSocket();
        } catch (Zygote.MethodAndArgsCaller caller) {
            caller.run();
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            zygoteServer.closeServerSocket();
            throw ex;
        }
    }
```