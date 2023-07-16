# 安全气囊如何实现？
## 前言
我们都知道，当 Andoird 程序发生未捕获的异常的时候，程序会直接 Crash 退出

而所谓安全气囊，是指在 Crash 发生时，可以捕获异常，触发兜底逻辑，在程序退出前做最后的抢救

接下来我们来看一下怎么实现一个安全气囊，以在 Crash 发生时做最后的抢救

## Java 层安全气囊
### Java 异常如何捕获
在实现安全气囊之前，我们先思考一个问题，像 bugly, sentry 这种库，是如何捕获异常并上传堆栈的呢？

要了解这个问题，我们首先要了解一下当异常发生时是怎么传播的

![](https://raw.githubusercontents.com/RicardoJiang/resource/main/2023/march/p7.png)

其实也很简单，主要分为以下几步

1. 当抛出异常时，通过`Thread.dispatchUncaughtException`进行分发
2. 依次由`Thread`，`ThreadGroup`，`Thread.getDefaultUncaughtExceptionHandler`处理
3. 在默认情况下，`KillApplicationHandler`会被设置`defaultUncaughtExceptionHandler`
4. `KillApplicationHandler`中会调用`Process.killProcess`退出应用

这就是异常发生时的传播路径，可以看出，如果我们通过`Thread.setDefaultUncaughtExceptionHandler`设置自定义处理器，就可以捕获异常做一些兜底操作了，其实 bugly 这些库也是这么做的

### 自定义异常处理器的问题

那么问题来了，如果我们设置了自定义处理器，在里面只做一些打印日志的操作，而不是退出应用，是不是就可以让 app 永不崩溃了呢？

答案当然是否定的，主要有以下两个问题

#### Looper 循环问题

![](https://raw.githubusercontents.com/RicardoJiang/resource/main/2023/march/p8.png)

我们知道，App 的运行在很大程序上依赖于 Handler 消息机制，`Handler` 不断的往 `MessageQueue` 中发送 `Message`，而`Looper`则死循环的不断从`MessageQueue`中取出`Message`并消费，整个 app 才能运行起来

而当异常发生时，`Looper.loop` 循环被退出了，事件也就不会被消费了，因此虽然 app 不会直接退出，但也会因为无响应发生 ANR

因此，当崩溃发生在主线程时，我们需要恢复一下`Looper.loop`

#### 主流程抛出异常问题
当我们在主淤积抛出异常时，比如在`onCreate`方法中，虽然我们捕获住了异常，但程序的执行也被中断了，界面的绘制可能无法完成，点击事件的设置也没有生效

这就导致了 app 虽然没有退出，但用户却无法操作的问题，这种情况似乎还不如直接 Crash 了呢

因此我们的安全气囊应该支持配置，只处理那些非主流程的操作，比如点击按钮触发的崩溃，或者一些打点等对用户无感知操作造成的崩溃

### 方案设计
为了解决上面提到的两个问题，我们的方案如下

![](https://raw.githubusercontents.com/RicardoJiang/resource/main/2023/march/p9.jpg)

主要分为以下几步：  
1. 注册自定义`DefaultUncaughtExceptionHandler`
2. 当异常发生是捕获异常
3. 匹配异常堆栈是否符合配置，如果符合则捕获，否则交给默认处理器处理
4. 判断异常发生时是否是主线程，如果是则重启`Looper`

### 代码实现
代码实现如下： 

```kotlin
    fun setUpJavaAirBag(configList: List<JavaAirBagConfig>) {
        val preDefaultExceptionHandler = Thread.getDefaultUncaughtExceptionHandler()
        // 设置自定义处理器
        Thread.setDefaultUncaughtExceptionHandler { thread, exception ->
            handleException(preDefaultExceptionHandler, configList, thread, exception)
            if (thread == Looper.getMainLooper().thread) {
            	// 重启 Looper
                while (true) {
                    try {
                        Looper.loop()
                    } catch (e: Throwable) {
                        handleException(
                            preDefaultExceptionHandler, configList, Thread.currentThread(), e
                        )
                    }
                }
            }
        }
    }

    private fun handleException(
        preDefaultExceptionHandler: Thread.UncaughtExceptionHandler,
        configList: List<JavaAirBagConfig>,
        thread: Thread,
        exception: Throwable
    ) {
    	// 匹配配置
        if (configList.any { isStackTraceMatching(exception, it) }) {
            Log.w("StabilityOptimize", "Java Crash 已捕获")
        } else {
            Log.w("StabilityOptimize", "Java Crash 未捕获，交给原有 ExceptionHandler 处理")
            preDefaultExceptionHandler.uncaughtException(thread, exception)
        }
    }
```

## Native 层安全气囊
通过上面的步骤，我们实现了一个 Java 层安全气囊，但是如果发生 Native 层崩溃时，程序还是会崩溃

那么我们能不能按照 Java 层安全气囊的思路，实现一个 Native 层的安全气囊？

### Native 异常如何捕获
Native 层异常是通过信号机制实现的

![](https://raw.githubusercontents.com/RicardoJiang/resource/main/2023/march/p10.png)

1. crash产生后，会在用户态阶段调用中断进入内核态
2. 在处理完内核操作，返回用户态时，会检查信号队列上是否有信号需要处理
3. 如果有信号需要处理，则会调用`sigaction`函数进行相应处理

那么如果我们通过注册信号处理函数`sigaction`设置自定义信号处理器，是不是可以实现跟 Java 安全气囊一样的效果？

需要注意的是，我们可以通过`sigaction`设置自定义信号处理器，但是`SIGKILL`与`SIGSTOP`信号我们是无法更改其默认行为的，如果我们设置了自定义信号处理器，没有退出 app，但错误实际还是产生了，当错误实在不可控时，系统还是会发送`SIGKILL/SIGSTOP`信号，这个时候还会导致我们 crash 时无法获取真正的堆栈，因此我们在自定义信号处理器时需要慎重

可以看出，要了解 Native 异常捕获，需要对 Linux 信号机制有一定了解，想了解更多的同学可以查看：[写给android开发的Linux 信号 - 上篇](https://juejin.cn/post/7196131493448613945)

### 代码实现
在了解了 Native 层异常处理的原理之后，我们通过自定义信号处理器来实现一个 Native 层的安全气囊，主要分为以下几步

1. 注册自定义信号处理器
2. 获取 Native 堆栈并与配置堆栈进行比较
3. 如果匹配上了则忽略相关崩溃，如果未匹配上则交给原信号处理器处理

```c++
extern "C" JNIEXPORT void JNICALL
Java_com_zj_android_stability_optimize_StabilityNativeLib_openNativeAirBag(
        JNIEnv *env,
        jobject /* this */,
        jint signal,
        jstring soName,
        jstring backtrace) {
    do {
        //...
        struct sigaction sigc;
        // 自定义处理器
        sigc.sa_sigaction = sig_handler;
        sigemptyset(&sigc.sa_mask);
        sigc.sa_flags = SA_SIGINFO | SA_ONSTACK | SA_RESTART;
        // 注册信号
        int flag = sigaction(signal, &sigc, &old);
    } while (false);
}

static void sig_handler(int sig, struct siginfo *info, void *ptr) {
	// 获取堆栈
    auto stackTrace = getStackTraceWhenCrash();
    // 与配置的堆栈进行匹配
    if (sig == airBagConfig.signal &&
        stackTrace.find(airBagConfig.soName) != std::string::npos &&
        stackTrace.find(airBagConfig.backtrace) != std::string::npos) {
        LOG("异常信号已捕获");
    } else {
    	// 没匹配上的交给原有处理器处理
        LOG("异常信号交给原有信号处理器处理");
        sigaction(sig, &old, nullptr);
        raise(sig);
    }
}
```

### 存在的问题
通过上面的步骤，其实 Native 层的安全气囊已经实现了，在 demo 中触发 Native Crash 可以被捕获到

但是信号处理函数必须是`async-signal-safe`和可重入的，理论上不应该在信号处理函数中做太多工作，比如`malloc`等函数都不是可重入的

而我们在信号处理函数中获取了堆栈，打印了日志，很可能会造成一些意料之外的问题

理论上我们可以在子线程获取堆栈，在信号处理函数中只需要发出信号就可以了，但我尝试在子线程中使用 unwind 获取堆栈，发现获取不到真正的堆栈，因此还存在一定的问题，有了解的大佬可以在评论区指点下

Native 层安全气囊的方案也可以看看@Pika 写的[https://github.com/TestPlanB/mooner](https://github.com/TestPlanB/mooner)，支持捕获 Android 基于“pthread_create” 产生的子线程中异常业务逻辑产生信号，导致的native crash

## 总结
本文主要介绍了Java 层与 Native 层安全气囊的实现方案与异常捕获原理，在一些非主流程的 Crash 发生时，通过安全气囊可以做一些最后的挽救，在降低崩溃率方面应该还是有一些应用场景的，希望本文对你有所帮助~

### 示例代码
本文所有源码可见：[https://github.com/RicardoJiang/android-performance](https://github.com/RicardoJiang/android-performance)