# 使用 KCP 打造更安全的 Gson 与更快的 Moshi
## 前言
众所周知，使用 Gson、Jackson 等框架反序列化 JSON 到 Kotlin 类时存在空安全问题和构造器默认值失效的问题，同时常用的 Gson, Moshi 等框架往往在功能上比较强大，全面，而在性能上却没有很明显的优势。本文将介绍如何使用 Kotlin 编译器插件打造更安全的 Gson 与更快的 Moshi。

### 空安全与默认值问题
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p5.png)

如上图所示，我们在使用 Gson 将 Json 数据反序列化为 `User` 对象时，会遇到空值安全问题和默认构造参数失效的问题。下面让我们来探讨一下为何会出现这样的情况。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p6.png)

- 当调用`Gson.fromJson`方法来反序列化对象时，首先会创建`Adapter`
- 在创建`Adapter`的过程时需要获取类的构建函数，如果存在无参构建函数会直接返回，否则会通过 UnSafe 的方式创建
- 当使用 UnSafe 方式创建时，就不会调用我们定义的主构造函数了，默认值自然也就失效了
- 在`Adapter`创建后，接下来就是在`Adapter`中通过`jsonReader`读取输入
- 在读取输入的过程中通过`field.set(target, fieldValue)`反射赋值，自然也就无法保证空安全了

### 性能问题
反序列化是 Android 开发中必备且非常高频的需求，因此一个高性能的反序列化框架就非常重要了，那么常用的 Gson 与 Moshi 等框架性能到底怎么样呢?

下面列出使用 Jetpack Microbenchmark 库测试常用反序列化框架的结果，具体测试过程可见：[常用 JSON 库性能对比](https://android-performance-optimization.github.io/practical/speed/json-serialization-speed/)

#### 多次运行测试结果
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/october/DeserializationSpeedMultiTimes.png)

从柱状图可以很明显的看出各个框架的速度对比

- Kotlin Serialization 看起来是最适合 Kotlin 的反序列化框架，在各个框架中表现最好
- JSONReader 与 JSONObject 在小数据上表现也不错，在大数据上 JSONReader 与其它框架相差不大，而 JSONObject 由于要将 InputStream 转化成 String，表现较差
- Moshi 与 Gson 在反序列化速度上差距不大，基本上是一个量级，但相比 JSONReader 等框架则明显较慢

#### 一次运行测试结果
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/october/DeserializationSpeedRunOnce.png)

可以看出，一次运行测试结果与多次运行测试结果明显不同

- JSONReader 与 JSONObject 相比其它框架优势明显，在冷启动场景使用这些系统原生 API 应该会有一定优势
- 在冷启动场景，Moshi 与 Kotlin Serialization 速度差不多，相比 Gson 则略慢

根据上述结果，可以看出 JSONReader 这些原生 API 在冷启动场景有极大优势，在经过充分优化后相比 Gson, Moshi 等库仍然有一定优势。因此，如果要实现更高性能的反序列化，使用 JSONReader 代替 Gson 与 Moshi，应该是个不错的选择。

但是 JSONReader 这些原生 API 使用起来较为麻烦，需要写很多模板代码，我们该如何优化呢？答案就是 Kudos。

## Kudos 是什么
Kudos 是 Kotlin utilities for deserializing objects 的缩写。它可以解决使用 Gson、Jackson 等框架反序列化 JSON 到 Kotlin 类时所存在的空安全问题和构造器默认值失效的问题，同时可以简化高性能的反序列化框架 JsonReader 的使用方式。

Kudos 已经在 Github 上开源，开源地址可见：[https://github.com/kanyun-inc/Kudos](https://github.com/kanyun-inc/Kudos)

## Kudos 使用
引入 Kudos 主要分为以下几步
### 1. 添加插件到 classpath
```kotlin
// 方式 1
// 传统方式，在根目录的 build.gradle.kts 中添加以下代码
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("com.kanyun.kudos:kudos-gradle-plugin:$latest_version")
    }
}

// 方式 2
// 引用插件新方式，在 settings.gradle.kts 中添加以下代码
pluginManagement {
    repositories {
        mavenCentral()
    }
    plugins {
        id("com.kanyun.kudos") version "$latest_version" apply false
    }
}
```

### 在项目中启用插件
```kotlin
plugins {
    // 启用 Kudos 插件. 
    // 为被 @Kudos 注解标注的类生成优化版本的无参构造器
    id("com.kanyun.kudos")
}

kudos {
    // 启用 Kudos.Gson. 为被 @Kudos 标注的类同时生成 @JsonAdapter 注解，并添加 kudos-gson 依赖.
    gson = true
    // 启用 Kudos.Jackson. 添加 kudos-jackson 依赖.
    jackson = true
    // 启用 Kudos.AndroidJsonReader. 添加 kudos-android-json-reader 依赖.
    androidJsonReader = true
}
```

我们可以在模块级别配置 Kudos 的功能开关

- 当设置`gson = true`时就会启用 Kudos.Gson，实现更安全的 Gson
- 当设置`jackson = true`时就会启用 Kudos.jackson，实现更安全的 Jackson
- 当设置`androidJsonReader = true`时就会启用 Kudos.AndroidJsonReader，简化 JsonReader 的使用

### 为特定类启动 Kudos 的支持
```kotlin
@Kudos
data class User(
    val id: Long, 
    val name: String,
    val age: Int = -1,
    val tel: String = ""
)

@Kudos(KUDOS_GSON)
data class User(
    val id: Long, 
    val name: String,
    val age: Int = -1,
    val tel: String = ""
)
```

- 对于需要添加 Kudos 解析支持的类型，直接添加`@Kudos`注解即可
- `@Kudos`注解的类的功能默认与模块配置一致，也可以添加参数例如`@Kudos(KUDOS_GSON)`实现针对该类只开启特定功能

## 原理解析：前置知识
### 编译器插件是什么？
Kotlin 的编译过程，简单来说就是将 Kotlin 源代码编译成目标产物的过程，具体步骤如下图所示：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p7.png)

Kotlin 编译器插件，通过利用编译过程中提供的各种 Hook 时机，让我们可以在编译过程中插入自己的逻辑，以达到修改编译产物的目的

Kotlin 编译器插件可以分为 Gradle 插件，编译器插件，IDE 插件三部分，如下图所示

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p8.png)

### K2 编译器与 K1 编译器
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p9.png)

#### 编译器前端
Kotlin 编译器可以分为编译器前端与编译器后端两部分，目前 Kotlin 编译器前端有两个版本，一个是老版本的 PSI 版本，一个是即将在 Kotlin 2.0 版本稳定的 FIR 版本。

PSI 即 Program Structure Interface，PSI 原本是 Intellij 平台对各类编程语言语法树的统一抽象，为了快速实现需求，Kotlin 团队在开发初期复用了 Intellij 平台已经有的技术积累。而从 Kotlin 1.6 版本开始，为了实现更好的编译器性能，Kotlin团队开始开发一款全新的前端编译器，即 K2 编译器的 FIR 前端。

因此当我们开发编译器插件时，在涉及编译器前端的修改时需要同时适配 K1 与 K2 两个版本。

#### 编译器后端
同时 Kotlin 作为一个多平台语言，可以将源代码编译成多个平台的目标代码，比如 Kotlin/Jvm 可以生成 Java 字节码，Kotlin/Js 可以生成 Javascript，Kotlin/Native 可以生成 .so 文件。

不同的编译器后端之间必然有不少逻辑可以共享，因此为了在不同的后端之间共享逻辑，降低支持新的语言特性的成本，同时方便后续扩展支持新的平台，Kotlin 编译器后端引入了 IR 这一中间层。

不过由于 Kotlin/Jvm 已经在 Kotlin 1.5 版本支持了 IR，Kotlin/Js 在 Kotlin 1.6 版本支持了 IR，而 Kotlin/Native 在一开始就支持 IR，因此我们在开发编译器插件时，涉及修改 IR 时不需要做什么额外的适配。

### 编译器扩展点
前面提到 Kotlin 编译器插件，通过利用编译过程中提供的各种扩展点，让我们可以在编译过程中插入自己的逻辑，以达到修改编译产物的目的，下面就介绍一下 Kudos 中使用到的扩展点以及对应的 API

| 扩展的类名                                | 编译阶段      | 功能说明                     | 用例                       |
|--------------------------------------|-----------|--------------------------|--------------------------|
| SyntheticResolveExtension            | 前端        | 解析生成的类，函数等，可用于给类添加接口或者方法 | Parcelize                |
| FirExtensionRegistrar                | 前端        | FIR 扩展，可用于提供代码声明信息与代码检查  | Parcelize                |
| StorageComponentContainerContributor | 前端        | 可用于编译期代码检查               | Compose                  |
| IrGenerationExtension                | 多平台 IR 后端 | 生成与修改 IR                 | Parcelize、Atomicfu、NoArg |

总得来说，在涉及到修改类或者函数的声明，例如函数的签名信息，给类添加接口等内容时，需要使用到编译器前端的扩展点；而当涉及到方法体的内部实现，类的私有属性等内容时，直接生成或修改 IR 即可。

## Kudos 原理解析
### 项目结构
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p10.png)

我们知道，Kudos 是一个 Kotlin 编译器插件，当然也可以分为 Gradle 插件，编译器插件，IDE 插件三部分，为了支持 Jackson，Kudos 也提供了 Maven 集成能力。

同时 Kudos 添加了对 Gson, Jackson, JsonReader 等框架的增强，针对这些框架的定制化代码封装在特定的 runtime 模块中，只有开启了对应的功能开关才会引入。

### Kudos 如何保证空安全与默认值？
前面介绍了 Kotlin 编译器插件相关的前置知识，那么 Kudos 到底是如何保证空安全与默认值的呢？

```kotlin
@Kudos
data class User(
    val id: Long, 
    val name: String,
    val age: Int = -1,
    val tel: String = ""
)
```

上面这段代码，在编译过后大致相当于

```kotlin
@Kudos
// 如果启用了 com.kanyun.kudos.gson 插件，则生成 @JsonAdapter 注解
@JsonAdapter(value = KudosReflectiveTypeAdapterFactory::class)
data class User(
    val id: Long, 
    val name: String,
    val age: Int = -1,
    val tel: String = ""
) : KudosValidator {
    constructor() { // 生成的默认无参构造器
        super() // 调用父类默认无参构造器
        init<User>() // 调用 User 类内部的 init 块（包括定义在内部的属性初始化）
        this.age = -1 // 使用主构造器的参数默认值初始化属性
        this.tel = "" // 使用主构造器的参数默认值初始化属性
    }
    
    // 生成的用于校验字段空安全的函数
    override fun validate(status: Map<String, Boolean>) {
        validateField("id", status)
        validateField("name", status)
    }
}
```

- Kudos 插件会生成默认无参构造器，在其中使用主构造器的参数默认值初始化属性，因此可以避免默认值失效。
- 当启用了 Kudos.gson 时，会给类添加`@JsonAdapter`注解，在自定义的`KudosReflectiveTypeAdapterFactory`中自定义反序列化逻辑
- Kudos 插件同样会给类添加`KudosValidator`接口与`validate`方法，在`validate`方法体中会给没有默认值的非空属性添加`validateField`调用
- 在`KudosReflectiveTypeAdapterFactory`中在反序列化完成后会调用`KudosValidator`的`validate`方法，以验证声明非空的属性是否不为空，否则会抛出异常

### Kudos 如何简化 JsonReader 的使用
JsonReader 使用起来麻烦的原因就在于要写很多模板代码，我们通过编译器插件来生成这些模板代码就可以解决 JsonReader 使用麻烦的问题。同时当我们开启 Kudos.AndroidJsonReader, 之前的保证默认值生效与空安全的代码也同样能生效。编译过后的代码如下所示：

```kotlin
@Kudos
data class User(
    val id: Long, 
    val name: String,
    val age: Int = -1,
    val tel: String = ""
) : KudosJsonAdapter {
    private var kudosFieldStatusMap: Map<String, Boolean> = hashMapOf()

    override fun fromJson(jsonReader: JsonReader): User {
        jsonReader.beginObject()
        while (jsonReader.hasNext()) {
            val tmp0 = jsonReader.nextName()
            if (jsonReader.peek() == JsonToken.NULL) {
                jsonReader.skipValue()
                continue
            }
            when {
                tmp0 == "id" -> {
                    <this>.id = jsonReader.nextLong()
                    <this>.kudosFieldStatusMap.put("id", <this>.id != null)
                }
                tmp0 == "name" -> {
                    <this>.name = jsonReader.nextString()
                    <this>.kudosFieldStatusMap.put("name", <this>.name != null)
                }
                // ...
                else -> {
                    jsonReader.skipValue()
                }
            }
        }
        jsonReader.endObject()
        validate(<this>.kudosFieldStatusMap)
        return this
    }
}
```

### kudos-compiler 的实现
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/december/p11.png)

前面我们介绍了 kudos-compiler 模块要实现的代码，那么作为提供核心能力的模块，kudos-compiler 具体又是如何实现的呢？

- KudosCommandLineProcessor 作为编译件插件的入口，注册 pluginId，同时接收命令行或者 gradle 插件传过来的参数
- KudosCompilerPluginRegistrar 的作用则是用于注册我们所需的各种扩展点
- KudosSyntheticResolveExtension 的作用是在符号引用解析时提供声明信息在符号引用解析时提供声明信息，比如给类添加`KudosValidator`接口与`validate`方法
- KudosFirExtensionRegistrar 的作用是在 K2 解析时提供声明信息与代码检查，与 KudosSyntheticResolveExtension 类似，区别在于两者适用的编译器前端版本不同
- KudosComponentContainerContributor 的作用是提供代码编译期检查
- KudosIrGenerationExtension 的作用是在编译产物中提供实现，比如`validate`与`fromJson`的方法体

## 性能数据
基于 Kudos 的工作机制不难想到，Kudos 的运行耗时会略微多于对应的 JSON 序列化框架。

使用 Jetpack Microbenchmark 库对 Kudos 与其对应的 JSON 序列化框架进行性能对比可以发现，Kudos.Gson 的耗时为 Gson 的 1.1-1.2 倍, Kudos.Jackson, KudosAndroidJsonReader 的情况类似，在可接受的范围之内。

### 多次运行测试结果
|                  | small json     | medium json    | large json     |
|------------------|----------------|----------------|----------------|
| Gson             | 412,375   ns   | 1,374,838   ns | 3,641,904   ns |
| Kudos-Gson       | 517,123   ns   | 1,686,568   ns | 4,311,910   ns |
| Jackson          | 1,035,010   ns | 1,750,709   ns | 3,450,974   ns |
| Kudos-Jackson    | 1,261,026   ns | 2,030,874   ns | 3,939,600   ns |
| JsonReader       | 190,302   ns   | 1,176,479   ns | 3,464,174   ns |
| Kudos-JsonReader | 215,974   ns   | 1,359,587   ns | 4,019,024   ns |

### 一次运行测试结果
|                  | small json      | medium json     | large json      |
|------------------|-----------------|-----------------|-----------------|
| Gson             | 3,974,219   ns  | 4,666,927   ns  | 8,271,355   ns  |
| Kudos-Gson       | 4,531,718   ns  | 6,244,479   ns  | 11,160,782   ns |
| Jackson          | 12,821,094   ns | 13,930,625   ns | 15,989,791   ns |
| Kudos-Jackson    | 13,233,750   ns | 15,674,010   ns | 18,641,302   ns |
| JsonReader       | 662,032   ns    | 2,056,666   ns  | 4,624,687   ns  |
| Kudos-JsonReader | 734,907   ns    | 2,362,010   ns  | 6,212,917   ns  |

## 如何学习 Kotlin 编译器插件
Kotlin 编译器插件目前还没有稳定，没有稳定的 API 与文档，那么我们该如何学习 Kotlin 编译器插件呢？

- 通过 Kotlin 源码学习：由于没有文档，Kotlin 源码实际上就是最新的文档，如果想要学习 KCP，可以先从 Kotlin 官方开发的插件，例如 Parcelize, NoArg 等开始
- 通过 AI 学习：当我们在开发编译器插件时，往往是知道想要生成的代码的样式，却由于没有 API 不知道如何实现，这种场景正是 AI 的用武之地了，我们可以提供给 AI 想要生成的代码示例，返回编译器插件的实现。
- 学习《深入实践 Kotlin 元编程》：这本书是目前少有的针对 Kotlin 编译器插件做了系统介绍的学习资料，除此之外，还介绍了反射，代码生成，程序静态分析，符号处理器等 Kotlin 元编程技术，想要深入学习 Kotlin 元编程的同学都可以了解下。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/august/p3.png)

## 总结
本文主要介绍了如何使用 Kudos ，以及 Kudos 到底是如何实现的，如果有任何问题，欢迎提出 Issue，如果对你有所帮助，欢迎点赞收藏 Star ~

### 开源地址
[https://github.com/kanyun-inc/Kudos](https://github.com/kanyun-inc/Kudos)



