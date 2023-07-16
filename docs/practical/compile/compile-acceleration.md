# 编译加速的8个实用技巧
## 前言
关于 `Android` 编译加速的文章相信大家都看过不少，但常常要么是好几年前写的，目前看来有些过时；要么介绍了一大堆配置，最后一实践发现并没有多大效果；要么就是大厂黑科技，但是没有开源。

今天我们就一起来看看，在2022年`AGP7.0`时代，除了传统的开启`build-cache`，打开并行编译，调整`Gradle`堆内存大小等常用手段之外，还有哪些可以落地的编译加速实用技巧

## 使用最新版本编译工具链
既然是2022年编译加速的实用技巧，首先就是要求编译工具链的版本比较新，后面介绍的技巧大部分也是新引入的特性

几乎每次更新时，`Android` 编译工具链都会得到一定性能上的优化或者是引入新的功能，因此我们应该及时跟进`Gradle`，`Android Gradle Plugin`和`Kotlin Gradle Plugin`等工具的更新，才能及时获得到相应的性能提升

## `Transform`迁移到`AsmClassVisitorFactory`
`Transform API`是`AGP1.5`就引入的特性，主要用于在`Android`构建过程中，在`Class`转`Dex`的过程中修改`Class`字节码。利用`Transform API`，我们可以拿到所有参与构建的`Class`文件，然后可以借助`ASM` 等字节码编辑工具进行修改，插入自定义逻辑。

国内很多团队都或多或少的用`AGP`的`Transform API`来搞点儿黑科技，比如无痕埋点，耗时统计，方法替换等。但是在`AGP7.0`中`Transform`已经被标记为废弃了，并且将在`AGP8.0`中移除

在`AGP7.0`之后，可以使用`AsmClassVisitorFactory`来做插桩，根据官方的说法，`AsmClassVisitoFactory`会带来约18%的性能提升，同时可以减少约5倍代码

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p6.webp)

`AsmClassVisitorFactory`之所以比`Transform`在性能上有优势，主要在于节省了`IO`的时间。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p7.webp)

如上图所示，多个`Transform`相互独立，都需要通过`IO`读取输入，修改字节码后将结果通过`IO`输出，供下一个`Transform`使用，如果每个`Transform`操作`IO`耗时+10s的话，各个`Transform`叠在一起，编译耗时就会呈线性增长

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p8.webp)

而使用`AsmClassVisitorFactory`则不需要我们手动进行`IO`操作，这是因为`AsmInstrumentationManager`中已经做了统一处理，只需要进行一次`IO`操作，然后交给`ClassVisitor`列表处理，完成后统一输出

通过这种方式，可以有效地减少`IO`操作，减少耗时。其实国内之前滴滴开源的`Booster`与字节开源的`Bytex`，都是通过这种思路来优化`Transform`性能的，现在官方终于跟进了

总得来说，`AsmClassVisitorFactory`在性能上与易用性上都有一定的提升，具体用法可参见：[Transform 被废弃，ASM 如何适配?](https://juejin.cn/post/7105925343680135198)

## KAPT 迁移到 KSP
注解处理器是`Android`开发中一种常用的技术，很多常用的框架比如`ButterKnife`，`ARouter`，`Glide`中都使用到了注解处理器相关技术

但是如果项目比较大的话，会很容易发现`KAPT`是拖慢编译速度的常见原因，这也是谷歌推出`KSP`取代`KAPT`的原因

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p9.webp)

从上面这张图其实就可以看出`KAPT`慢的原因了，`KAPT` 通过与 `Java` 注解处理基础架构相结合，让大部分`Java`语言注解处理器能够在`Kotlin`中开箱即用。

为此，`KAPT` 首先需要将 `Kotlin` 代码编译成 `JavaStubs`，这些`JavaStubs`中保留了`Java`注释处理器关注的信息。

这意味着编译器必须多次解析程序中的所有符号 (一次生成`JavaStubs`，另一次完成实际编译)，但是生成`JavaStubs`的过程是非常耗时的，往往生成`Java Stubs`的时间比`APT`真正处理注解的时间要长

而`KSP`不需要生成`JavaStubs`，而是作为`Kotlin`编译器插件运行。它可以直接处理`Kotlin`符号，而不需要依赖`Java`注解处理基础架构。

因为`KSP`相比`KAPT`少了生成`JavaStubs`的过程，因此通常可以得到100%以上的速度提升。关于`KSP`的具体使用方法可参见：[使用 Kotlin Symbol Processing 1.0 缩短 Kotlin 构建时间](https://mp.weixin.qq.com/s/9WVWvyEj7YE6QAvIgyAF_Q)

### 迁移方案
目前`KSP`已经发布了稳定版了，像`Room`，`Moshi`等库也已经做了适配，对于这些已经适配了的库，我们可以直接迁移。

但还是有一些常用的库比如`Glide`，`ARouter`还没有做适配，这些库是我们移除`KAPT`最大的障碍。

下面给出一些还不支持`KSP`的库的过渡迁移方法

1. `KAPT`一般就是用来生成代码，像`Glide`这种生成的代码比较稳定的库(通常只有几个`@GlideModule`)，可以直接把生成的代码从`build`目录拷贝到项目中来，然后移除`KAPT`，后续如果有新的`@GlideModule`再更新下生成的文件（当然这样可能不太方便，只是一种过渡的方式，等待`Glide`官方更新）
2. 对于`ARouter`这种生成代码不断增加的库(不断有新的`@ARouter`注解)，上面的方式就不太适用了。考虑到`ARoutr`已经很久没有更新了，可以考虑迁移到一个不使用`KAPT`的新的路由库

**更新：** `Glide`最新版本已经支持了`KSP`，可以直接升级接入了

## 开启`Configuration Cache`
我们知道，`Gradle` 的生命周期可以分为大的三个部分：初始化阶段（`Initialization Phase`)，配置阶段（`Configuration Phase`），执行阶段（`Execution Phase`），如下图所示：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p10.webp)

在任务执行阶段，`Gradle`提供了多种方式实现`Task`的缓存与重用（如`up-to-date`检测，增量编译，`build-cache`等）

除了任务执行阶段，任务配置阶段有时也比较耗时，目前`AGP`也支持了配置阶段缓存`Configuration Cache`,它可以缓存配置阶段的结果，当脚本没有发生改变时可以重用之前的结果

在越大的项目中配置阶段缓存的收益越大，`module`比较多的项目可能每次执行都要先配置20到30秒，尤其是增量编译时，配置的耗时可能都跟执行的耗时差不多了，而这正是`configuration-cache`的用武之地

目前`Configuration-cache`还是实验特性，如果你想要开启的话可以在`gradle.properties`中添加以下代码

```
# configuration cache
org.gradle.unsafe.configuration-cache=true
org.gradle.unsafe.configuration-cache-problems=warn
```

开启了`Configuration cache`之后效果还是比较明显的，如果构建脚本没有发生变化可以直接跳过配置阶段

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p10.webp)

`Android`官方给出了一个开启`Configuration cache`前后的对比，可以看出在这个`benchmark`中可以得到约30%的提升(当然是在配置阶段耗时占比越高的时候效果越明显，全量编译时应该不会有这样的比例)

### `Configuration Cache`适配
当然打开`Configuration Cache`之后可能会有一些适配问题，如果是第三方插件，发现常用插件出现不支持的情况，可先搜索是否有相同的问题已经出现并修复

如果是项目中自定义`Task`不支持的话，还需要适配一下`Configuration Cache`，适配`Configuration Cache`的核心思路其实很简单：不要在`Task`执行阶段调用外部不可序列化的对象(比如`Project`与`Variant`)

不过如果你的项目中自定义`Task`比较多的话，适配`Configuration Cache`可能是个体力活，比如 `AGP` 兼容 `Configuration Cache` 就修了 400 多个 `ISSUE`

如需详细了解配置缓存，请参阅[配置缓存深度解析](https://medium.com/androiddevelopers/configuration-caching-deep-dive-bcb304698070)和[有关配置缓存的 Gradle 文档](https://docs.gradle.org/current/userguide/configuration_cache.html)

## 移除`Jetifier`	
`Jetifier`是`android support`包迁移到`androidX`的工具，当你在项目中启动用`Jetifier`时 ，`Gradle`插件会在构建时将三方库里的`Support`转换成`AndroidX`，因此会对构建速度产生影响。

同时`Jetfier`也会对`sync`耗时产生比较大的影响，详情可见B站大佬的分析：[哔哩哔哩 Android 同步优化•Jetifier](https://mp.weixin.qq.com/s/EExWHagW8f1s2hDIjYmjKg)

`Jetifier`在`AndroidX`刚出现时是一个非常实用的工具，可以帮助我们快速的迁移到`AndroidX`。但是到了2022年，相信绝大多数库都已经迁移到了`AndroidX`，`Jetifier`的历史使命可以说已经完成了，因此是时候移除`Jetifier`了

### 检测不支持`Jetifier`的库
`AGP7.0`已经提供了工具供我们检查每个`module`能否移除`Jetifier`，直接运行`./gradlew checkJetifier`即可，通过以下命令检查所有`module`的`Jetifier`使用情况

```
task checkJetifierAll(group: "verification") { }
 
subprojects { project ->
    project.tasks.whenTaskAdded { task ->
        if (task.name == "checkJetifier") {
            checkJetifierAll.dependsOn(task)
        }
    }
}
```

通过运行`./gradlew checkJetifierAll`就可以打印出所有`module`的`Jetifier`使用情况

### 迁移方案
在明确了哪些库还不支持`Jetifier`之后，可以一步步开始迁移了

1. 检测库有没有已经支持了`androidX`的最新版本， 如果有直接升级即可
2. 如果有源码，添加`android.useAndroidX = true`，迁移到`AndroidX`，然后升级所有的依赖，发布个新版本就可以了。     
3. 如果你没有源码，或对于你的项目来说，它太老了。你可以用[jetifier-standalone](https://developer.android.com/studio/command-line/jetifier)命令行工具把`AAR/JAR`转成`jetified`之后的`AAR/JAR`。这个命令行的转换效果和你在代码里开启`android.enableJetifier`的效果是一样的。命令行如下：


```
// https://developer.android.com/studio/command-line/jetifier    
./jetifier-standalone -i <source-library> -o <output-library>  
```

## 关闭`R`文件传递	
在 `apk` 打包的过程中，`module` 中的 `R` 文件采用对依赖库的`R`进行累计叠加的方式生成。如果我们的 `app` 架构如下：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p12.webp)

编译打包时每个模块生成的`R`文件如下：

```
1. R_lib1 = R_lib1;
2. R_lib2 = R_lib2;
3. R_lib3 = R_lib3;
4. R_biz1 = R_lib1 + R_lib2 + R_lib3 + R_biz1(biz1本身的R)
5. R_biz2 = R_lib2 + R_lib3 + R_biz2(biz2本身的R)
6. R_app = R_lib1 + R_lib2 + R_lib3 + R_biz1 + R_biz2 + R_app(app本身R)
```

可以看出各个模块的R文件都会包含上层组件的`R`文件内容，这不仅会带来包体积问题，也会给编译速度带来一定的影响。比如我们在`R_lib1`中添加了资源，所有下游模块的`R`文件都需要重新编译。

1. 关闭`R`文件传递可以通过编译避免的方式获得更快的编译速度
2. 关闭`R`文件传递有助于确保每个模块的`R`类仅包含对其自身资源的引用，避免无意中引用其他模块资源，明确模块边界。
3. 关闭`R`文件传递也可以减少很大一部分包体积与`dex`数量

### 迁移方案
从 `Android Studio Bumblebee` 开始，新项目的非传递 `R` 类默认处于开启状态。即`gradle.properties`文件中都开启了如下标记

```
android.nonTransitiveRClass=true
```

对于使用早期版本的 `Studio` 创建的项目，您可以依次前往 `Refactor > Migrate to Non-transitive R Classes`，将项目更新为使用非传递 `R` 类。

## 开启`Kotlin`跨模块增量编译	
使用组件化多模块开发的同学都有经验，当我们修改底层模块(比如`util`模块)时，所有依赖于这个模块的上层模块都需要重新编译，`Kotlin`的增量编译在这种情况往往是不生效的，这种时候的编译往往非常耗时

在`Kotlin 1.7.0`中，`Kotlin`编译器对于跨模块增量编译也做了支持，并且与`Gradle`构建缓存兼容，对编译避免的支持也得到了改进。这些改进减少了模块和文件重新编译的次数，让整体编译更加迅速

### 优化效果
首先来看下`Kotlin`官方的数据，以下基准测试结果是在`Kotlin`项目中的`kotlin-gradle-plugin`模块上测得：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p13.webp)

可以看出，当缓存命中时有86%到96%的加速效果，当缓存没有命中时也有26%的加速效果

我在项目中开启后实测效果也很不错，修改一个底层模块，在特性开启前需要耗时4分钟左右，开启后增量编译耗时减少到30到40s，加速约85%

### 如何开启
在 `gradle.properties` 文件中设置以下选项即可使用新方式进行增量编译：

```
kotlin.incremental.useClasspathSnapshot=true // 开启跨模块增量编译
kotlin.build.report.output=file // 可选，启用构建报告
```

可以看出，开启步骤还是非常简单的，关于`Kotlin`跨模块增量编译的原理可参见：[Kotlin 增量编译的新方式](https://blog.jetbrains.com/zh-hans/kotlin/2022/07/a-new-approach-to-incremental-compilation-in-kotlin/)

对于增量编译，稳定性和可靠性至关重要。有时增量编译总会失效，`Kotlin 1.7`同样支持为编译任务创建编译报告，报告包含不同编译阶段的持续时间以及无法使用增量编译的原因，可以帮助你定位为什么增量编译失效了

关于编译报告的启用与使用可见：[隆重推出 Kotlin 构建报告](https://blog.jetbrains.com/zh-hans/kotlin/2022/07/introducing-kotlin-build-reports/)

## 升级你的电脑配置
除了上述的软件方向的一系列优化，也可以从硬件方向进行优化，也就是升级你的电脑配置

个人感觉影响编译速度的关键基本配置如下：

- `CPU`：2022年了，最好直接上`M1`吧，的确要快不少，相信大家应该看到过一些说换`M1`后编译速度变快的帖子
- 内存：至少要16G，有条件建议上32G，对于一些大型项目，内存甚至比`CPU`更重要，因为`Gradle`守护进程占用的内存可以非常大
- 硬盘：必须是`SSD`固态硬盘，256G勉强够用，最好是512G，`Gradle`构建缓存(`build-cache`)占用的空间也挺大的

从硬件方向入手，有时也可以得到不错的优化效果，**充钱是真的可以变强的**

## 总结
本文主要介绍了编译加速的8个实用技巧，有的接入起来非常简单，有的则需要一定的适配成本，但都是可以落地的并且有一定效果的编译加速技巧

总得来说，为了充分利用最新的优化技巧与各种新功能，我们应该及时跟进`android`编译工具链的更新

如果本文对你有所帮助，欢迎点赞收藏~

### 更多
[Android官方文档 - 优化构建速度](https://developer.android.com/studio/build/optimize-your-build?hl=zh-cn)        
[How we reduced our Gradle build times by over 80%](https://proandroiddev.com/how-we-reduced-our-gradle-build-times-by-over-80-51f2b6d6b05b)         
[10 ideas to improve your Gradle build times](https://blog.dipien.com/10-great-ideas-to-improve-your-gradle-build-times-2a6b281c69c6)
