# nowinandroid 构建脚本学习
## 前言
[nowinandroid](https://github.com/android/nowinandroid) 项目是谷歌开源的示例项目，它遵循 Android 设计和开发的最佳实践，并旨在成为开发人员的有用参考

这个项目在架构演进，模块化方案，单元测试，Jetpack Compose，启动优化等多个方面都做了很好的示例，的确是一个值得学习的好项目

今天我们来学习一下 nowinandroid 项目的构建脚本，看一下都有哪些值得学习的地方

## gradle.properties 中的配置
要看一个项目的构建脚本，我们首先看一下 gradle.properties

```
# Enable configuration caching between builds.
org.gradle.unsafe.configuration-cache=true

android.useAndroidX=true

# Non-transitive R classes is recommended and is faster/smaller
android.nonTransitiveRClass=true

# Disable build features that are enabled by default,
# https://developer.android.com/studio/releases/gradle-plugin#buildFeatures
android.defaults.buildfeatures.buildconfig=false
android.defaults.buildfeatures.aidl=false
android.defaults.buildfeatures.renderscript=false
android.defaults.buildfeatures.resvalues=false
android.defaults.buildfeatures.shaders=false
```

可以看出，nowinandroid 项目主要做了以下几个配置

1. 开启配置阶段缓存
2. 开启 `androidX`，并且移除了 `Jetifier`
3. 关闭 `R` 文件传递
4. 关闭 `build features`

前面3个配置之前都介绍过，我们来看一下关闭 `build features`

 AGP 4.0.0 引入了一种新方法来控制您要启用和停用哪些构建功能，如`ViewBinding`，`BuildConfig`。

我们可以在 gradle.properties 中全局开启或关闭某些功能，也可以在模块级 build.gradle 文件中为每个模块设置相应的选项，如下所示：

```

android {
    // The default value for each feature is shown below. You can change the value to
    // override the default behavior.
    buildFeatures {
        // Determines whether to generate a BuildConfig class.
        buildConfig = true
        // Determines whether to support View Binding.
        // Note that the viewBinding.enabled property is now deprecated.
        viewBinding = false
        // Determines whether to support Data Binding.
        // Note that the dataBinding.enabled property is now deprecated.
    }
}
```

通过停用不需要的构建可能，可以提升我们的构建性能，比如我们最熟悉的`BuildConfig`，每个模块都会生成这样一个类，但其实我们在绝大多数情况下是用不到的，因此其实可以将其默认关闭（在 AGP 8.0 中 BuildConfig 生成已经变成默认关闭了）


## 自动安装 git hook
有时我们会添加一些 git hook，用于在代码提交或者 push 时做一些检查

但使用 git hook 的一个问题在于，每次拉取新项目之后，都需要手动安装一下 git hook，这一点常常容易被忘记

那么有没有什么办法可以自动安装 git hook 呢？nowinandroid 项目提供了一个示例

```
// settings.gradle.kts

val prePushHook = file(".git/hooks/pre-push")
val commitMsgHook = file(".git/hooks/commit-msg")
val hooksInstalled = commitMsgHook.exists()
    && prePushHook.exists()
    && prePushHook.readBytes().contentEquals(file("tools/pre-push").readBytes())

if (!hooksInstalled) {
    exec {
        commandLine("tools/setup.sh")
        workingDir = rootProject.projectDir
    }
}
```

其实原理很简单，在`settings.gradle.kts`中添加以上代码，这样在 Gradle 同步时，就会自动判断 git hook 有没有被安装，如果没有被安装则自动安装

## 使用 includeBuild 而不是 buildSrc
```
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

为了支持在不同的模块间共享构建逻辑，此前我们常常会添加一个 buildSrc 模块

但是 buildSrc 模块的问题在于每次发生修改都会导致项目的绝大多数缓存失效，从而导致构建速度变得极慢

因此官方现在更推荐我们使用 includeBuild，比如 nowinandroid 的构建逻辑就通过 includeBuild 放在了 `build-logic` 目录

## 如何复用 build.gradle 代码
其实我们项目中的各个模块的 build.gradle 中的代码，大部分是重复的，做的都是一些重复的配置，当要修改时就需要一个一个去修改了

nowinandroid 通过抽取重复配置的方式大幅度的减少了 build.gradle 中的代码，如下所示

```
plugins {
    id("nowinandroid.android.feature")
    id("nowinandroid.android.library.compose")
    id("nowinandroid.android.library.jacoco")
}

android {
    namespace = "com.google.samples.apps.nowinandroid.feature.author"
}

dependencies {
    implementation(libs.kotlinx.datetime)
}
```

这是 nowinandroid 的一个 feature 模块，可以看出除了每个模块不同的`namespace`与各个模块的依赖之外，其他的内容都抽取到`nowinandroid.android.feature`等插件中去了，而这些插件的代码都存放在`build-logic` 目录中，通过 includeBuild 引入，大家可自行查看

总得来说，通过这种方式可以大幅减少重复配置代码，当配置需要迁移时也更加方便

## 使用 Version Catalog
在 build.gradle 中添加依赖有以下几个痛点

1. 项目依赖统一管理，在单独文件中配置
2. 不同Module中的依赖版本号统一
3. 添加依赖时支持代码提示

针对这几种需求，Gradle7.0 推出了一个新的特性，使用 Version Catalog 统一依赖版本，它支持以下特性：

- 对所有 module 可见，可统一管理所有module的依赖
- 支持声明依赖bundles,即总是一起使用的依赖可以组合在一起
- 支持版本号与依赖名分离，可以在多个依赖间共享版本号
- 支持在单独的libs.versions.toml文件中配置依赖
- 支持代码提示(仅 kts)

![](https://raw.githubusercontents.com/RicardoJiang/resource/main/2022/november/p9.jpg)

noinandroid 中目前已经全面启用了 Version Catalog，如上所示，统一依赖版本，支持代码提示，体验还是不错的

关于 Version Catalog 的具体使用可以查看：[【Gradle7.0】依赖统一管理的全新方式，了解一下~](https://juejin.cn/post/6997396071055900680)

## 代码格式检查
nowinandroid 作为一个开源项目，不可避免地会有第三方贡献一些代码，因此也需要在代码合并前做一些格式检查，保证代码风格的统一

nowinandroid 通过 spotless 来检查代码格式，主要是通过两种方式触发    

1. 通过上面提到的 git hook，在代码 push 时触发检查
2. 通过 github workflow，在代码 push 到 main 分支时触发检查

上面两种方式都会调用以下命令

```
./gradlew spotlessCheck --init-script gradle/init.gradle.kts --no-configuration-cache --stacktrace
```

可以看出，这里主要是执行 spotlessCheck 任务，并且指定了 init-script，我们来看一下 init.gradle.kts 里面做了什么

```
// init.gradle.kts
rootProject {
    subprojects {
        apply<com.diffplug.gradle.spotless.SpotlessPlugin>()
        extensions.configure<com.diffplug.gradle.spotless.SpotlessExtension> {
            kotlin {
                target("**/*.kt")
                targetExclude("**/build/**/*.kt")
                ktlint(ktlintVersion).userData(mapOf("android" to "true"))
                licenseHeaderFile(rootProject.file("spotless/copyright.kt"))
            }
            format("kts") {
                target("**/*.kts")
                targetExclude("**/build/**/*.kts")
                // Look for the first line that doesn't have a block comment (assumed to be the license)
                licenseHeaderFile(rootProject.file("spotless/copyright.kts"), "(^(?![\\/ ]\\*).*$)")
            }
            format("xml") {
                target("**/*.xml")
                targetExclude("**/build/**/*.xml")
                // Look for the first XML tag that isn't a comment (<!--) or the xml declaration (<?xml)
                licenseHeaderFile(rootProject.file("spotless/copyright.xml"), "(<[^!?])")
            }
        }
    }
}
```

可以看出，这里指定了对于 kotlin , kts , xml 等文件的格式要求，比如 kotlin 代码需要遵守 ktlint 规范，并且文件开头必须是 license 声明

## 自定义 lint 检查
除了代码风格的统一，nowinandroid 项目还自定义了一些 lint 检查，跟 spoltess 一样，也是通过 git hook 与 github workflow 两种方式触发，两种方式都会触发以下代码

```
./gradlew lintDemoDebug --stacktrace
```

nowinandroid 中有一个自定义的 lint 模块，自定义 lint 规则就定义在这里，如下所示：

```kotlin
class DesignSystemDetector : Detector(), Detector.UastScanner {

    override fun createUastHandler(context: JavaContext): UElementHandler {
        return object : UElementHandler() {
            override fun visitCallExpression(node: UCallExpression) {
                val name = node.methodName ?: return
                val preferredName = METHOD_NAMES[name] ?: return
                reportIssue(context, node, name, preferredName)
            }

            override fun visitQualifiedReferenceExpression(node: UQualifiedReferenceExpression) {
                val name = node.receiver.asRenderString()
                val preferredName = RECEIVER_NAMES[name] ?: return
                reportIssue(context, node, name, preferredName)
            }
        }
    }

    companion object {
        @JvmField
        val ISSUE: Issue = Issue.create(
            id = "DesignSystem",
            briefDescription = "Design system",
            explanation = "This check highlights calls in code that use Compose Material " +
                "composables instead of equivalents from the Now in Android design system " +
                "module."
        )

        // Unfortunately :lint is a Java module and thus can't depend on the :core-designsystem
        // Android module, so we can't use composable function references (eg. ::Button.name)
        // instead of hardcoded names.
        val METHOD_NAMES = mapOf(
            "MaterialTheme" to "NiaTheme",
            "Button" to "NiaFilledButton",
            "OutlinedButton" to "NiaOutlinedButton",
            // ...
        )
        val RECEIVER_NAMES = mapOf(
            "Icons" to "NiaIcons"
        )

        fun reportIssue(
            context: JavaContext, node: UElement, name: String, preferredName: String
        ) {
            context.report(
                ISSUE, node, context.getLocation(node),
                "Using $name instead of $preferredName"
            )
        }
    }
}
```

总得来说，这个自定义规则是检查是否使用了 Compose 的默认 Material 组件而没有使用 nowinandroid 封装好的组件，如果检查不通过则会抛出异常，提醒开发者修改

## 总结
本文主要介绍了 nowinandroid 项目构建脚本中的一系列小技巧，具体包括以下内容

1. gradle.properties 中的配置
2. 自动安装 git hook
3. 使用 includeBuild 而不是 buildSrc
4. 如何复用 build.gradle 代码
5. 使用 Version Catalog
6. 代码格式检查
7. 自定义 lint 检查

希望对你有所帮助~

### 项目地址
[https://github.com/android/nowinandroid](https://github.com/android/nowinandroid)

