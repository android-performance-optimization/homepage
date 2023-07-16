# 落地 Kotlin 代码规范，DeteKt 了解一下
## 前言
各个团队多少都有一些自己的代码规范，但制定代码规范简单，困难的是如何落地。如果完全依赖人力`Code Review`难免有所遗漏。

这个时候就需要通过静态代码检查工具在每次提交代码时自动检查，本文主要介绍如何使用`DeteKt`落地`Kotlin`代码规范，主要包括以下内容

1. 为什么使用`DeteKt`?
2. `IDE`接入`DeteKt`插件
3. `CLI`命令行方式接入`DeteKt`
4. `Gradle`方式接入`DeteKt`
5. 自定义`Detekt`检测规则
6. `Github Action`集成`Detekt`检测

## 为什么使用`DeteKt`?
说起静态代码检查，大家首先想起来的可能是`lint`，相比`DeteKt`只支持`Kotlin`代码，`lint`不仅支持`Kotlin`，`Java`代码，也支持资源文件规范检查，那么我们为什么不使用`Lint`呢？

在我看来，`Lint`在使用上主要有两个问题：   

1. 与`IDE`集成不够好，自定义`lint`规则的警告只有在运行`./gradlew lint`后才会在`IDE`上展示出来，在`clean`之后又会消失
2. `lint`检查速度较慢，尤其是大型项目，只对增量代码进行检查的逻辑需要自定义

而`DeteKt`提供了`IDE`插件，开启后可直接在`IDE`中查看警告，这样可以在第一时间发现问题，避免后续检查发现问题后再修改流程过长的问题

同时`Detekt`支持`CLI`命令行方式接入与`Gradle`方式接入，支持只检查新增代码，在检查速度上比起`lint`也有一定的优势

## `IDE`接入`DeteKt`插件
如果能在`IDE`中提示代码中存在的问题，应该是最快发现问题的方式，`DeteKt`也贴心的为我们准备了插件，如下所示：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p14.webp)

主要可以配置以下内容：    
1. `DeteKt`开关
2. 格式化开关，`DeteKt`直接使用了`ktlint`的规则
3. `Configuration file`：规则配置文件，可以在其中配置各种规则的开关与参数，默认配置可见：[default-detekt-config.yml](https://github.com/detekt/detekt/blob/main/detekt-core/src/main/resources/default-detekt-config.yml)
4. `Baseline file`：基线文件，跳过旧代码问题，有了这个基线文件，下次扫描时，就会绕过文件中列出的基线问题，而只提示新增问题。
5. `Plugin jar`: 自定义规则`jar`包，在自定义规则后打出`jar`包，在扫描时就可以使用自定义规则了

`DeteKt IDE`插件可以实时提示问题(包括自定义规则)，如下图所示，我们添加了自定义禁止使用`kae`的规则：  

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p15.webp)

对于一些支持自动修复的格式问题，`DeteKt`插件支持自动格式化，同时也可以配置快捷键，一键自动格式化，如下所示：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p16.webp)

## `CLI`命令行方式接入`DeteKt`
`DeteKt`支持通过`CLI`命令行方式接入，支持只检测几个文件，比如本次`commit`提交的文件

我们可以通过如下方式，下载`DeteKt`的`jar`然后使用

```
curl -sSLO https://github.com/detekt/detekt/releases/download/v1.22.0-RC1/detekt-cli-1.22.0-RC1.zip
unzip detekt-cli-1.22.0-RC1.zip
./detekt-cli-1.22.0-RC1/bin/detekt-cli --help
```

`DeteKt CLI`支持很多参数，下面列出一些常用的，其他可以参见：[Run detekt using Command Line Interface](https://detekt.dev/docs/gettingstarted/cli)

```
Usage: detekt [options]
  Options:
    --auto-correct, -ac
      支持自动格式化的规则自动格式化，默认为false
      Default: false
    --baseline, -b
      如果传入了baseline文件，只有不在baseline文件中的问题才会掘出来
    --classpath, -cp
      实验特性：传入依赖的class路径和jar的路径，用于类型解析
    --config, -c
      规则配置文件，可以配置规则开关及参数
    --create-baseline, -cb
      创建baseline，默认false，如果开启会创建出一个baseline文件，供后续使用
    --input, -i
      输入文件路径，多个路径之间用逗号连接
    --jvm-target
      EXPERIMENTAL: Target version of the generated JVM bytecode that was 
      generated during compilation and is now being used for type resolution 
      (1.6, 1.8, 9, 10, 11, 12, 13, 14, 15, 16 or 17)
      Default: 1.8
    --language-version
      为支持类型解析，需要传入java版本
    --plugins, -p
      自定义规则jar路径，多个路径之间用,或者;连接
```

在命令行可以直接通过如下方式检查

```
java -jar /path/to/detekt-cli-1.21.0-all.jar # detekt-cli-1.21.0-all.jar所在路径
-c /path/to/detekt_1.21.0_format.yml # 规则配置文件所在路径
--plugins /path/to/detekt-formatting-1.21.0.jar # 格式化规则jar，主要基于ktlint封装
-ac # 开启自动格式化
-i $FilePath$ # 需要扫描的源文件，多个路径之间用,或者;连接
```

通过如上方式进行代码检查速度是非常快的，根据经验来说一般就是几秒之内可以完成，因此我们完成可以将`DeteKt`与`git hook`结合起来，在每次提交`commit`的时候进行检测，而如果是一些比较耗时的工具比如`lint`，应该是做不到这一点的

### 类型解析

上面我们提到了，`DeteKt`的`--classpth`参数与`--language-version`参数，这些是用于类型解析的。

类型解析是`DeteKt`的一项功能，它允许 `Detekt` 对您的 `Kotlin` 源代码执行更高级的静态分析。

通常，`Detekt` 在编译期间无法访问编译器语义分析的结果，我们只能获取`Kotlin`源代码的抽象语法树，却无法知道语法树上符号的语义，这限制了我们的检查能力，比如我们无法判断符号的类型，两个符号究竟是不是同一个对象等

通过启用类型解析，`Detekt` 可以获取`Kotlin`编译器语义分析的结果，这让我们可以自定义一些更高级的检查。

而要获取类型与语义，当然要传入依赖的`class`，也就是`classpath`，比如`android`项目中常常需要传入`android.jar`与`kotlin-stdlib.jar`

## `Gradle`方式接入`DeteKt`
`CLI`方式检测虽然快，但是需要手动传入`classpath`，比较麻烦，尤其是有时候自定义规则需要解析我们自己的类而不是`kotlin-stdlib.jar`中的类时，那么就需要将项目中的代码的编译结果传入作为`classpath`了，这样就更麻烦了

`DeteKt`同样支持`Gradle`插件方式接入，这种方式不需要我们另外再配置`classpath`，我们可以将`CLI`命令行方式与`Gradle`方式结合起来，在本地通过`CLI`方式快速检测，在`CI`上通过`Gradle`插件进行完整的检测

### 接入步骤
```
// 1. 引入插件
plugins {
    id("io.gitlab.arturbosch.detekt").version("[version]")
}

repositories {
    mavenCentral()
}

// 2. 配置插件
detekt {
    config = files("$projectDir/config/detekt.yml") // 规则配置
    baseline = file("$projectDir/config/baseline.xml") // baseline配置
    parallel = true
}

// 3. 自定义规则
dependencies {
    detektPlugins "io.gitlab.arturbosch.detekt:detekt-formatting:1.21.0"
    detektPlugins project(":customRules")
}

// 4. 配置 jvmTarget
tasks.withType(Detekt).configureEach {
    jvmTarget = "1.8"
}
// DeteKt Task用于检测，DetektCreateBaselineTask用于创建Baseline
tasks.withType(DetektCreateBaselineTask).configureEach {
    jvmTarget = "1.8"
}

// 5. 只分析指定文件
tasks.withType<io.gitlab.arturbosch.detekt.Detekt>().configureEach {
    // include("**/special/package/**") //  只分析 src/main/kotlin 下面的指定目录文件
    exclude("**/special/package/internal/**") // 过滤指定目录
}

```

如上所示，接入主要需要做这么几件事：  

1. 引入插件
2. 配置插件，主要是配置`config`与`baseline`，即规则开关与老代码过滤
3. 引入`detekt-formatting`与自定义规则的依赖
4. 配置`JvmTarget`，用于类型解析，但不用再配置`classpath`了。
5. 除了`baseline`之外，也可以通过`include`与`exclude`的方式指定只扫描指定文件的方式来实现增量检测

通过以上方式就接入成功了，运行`./gradlew detektDebug`就可以开始检测了，扫描结果可在终端直接查看，并可以直接定位到问题代码处，也可以在`build/reprots/`路径下查看输出的报告文件：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p17.webp)

## 自定义`Detekt`检测规则
要落地自己制定的代码规范，不可避免的需要自定义规则，当然我们首先要看下`DeteKt`自带的规则，是否已经有我们需要的，只需把开关打开即可.

### `DeteKt`自带规则
`DeteKt`自带的规则都可以通过开关配置，如果没有在 `Detekt` 闭包中指定 `config` 属性，`detekt` 会使用默认的规则。这些规则采用 `yaml` 文件描述，运行 `./gradlew detektGenerateConfig` 会生成 `config/detekt/detekt.yml` 文件，我们可以在这个文件的基础上制定代码规范准则。

`detekt.yml` 中的每条规则形如：

```
complexity: # 大类
  active: true
  ComplexCondition: # 规则名
    active: true  # 是否启用
    threshold: 4  # 有些规则，可以设定一个阈值
# ...
```

更多关于配置文件的修改方式，请参考[官方文档-配置文件](https://detekt.dev/docs/introduction/configurations)

`Detekt` 的规则集划分为 9 个大类，每个大类下有具体的规则：

规则大类           | 说明                               |
| -------------- | -------------------------------- |
| comments       | 与注释、文档有关的规范检查                    |
| complexity     | 检查代码复杂度，复杂度过高的代码不利于维护            |
| coroutines     | 与协程有关的规范检查                       |
| empty-blocks   | 空代码块检查，空代码应该尽量避免                 |
| exceptions     | 与异常抛出和捕获有关的规范检查                  |
| formatting     | 格式化问题，detekt直接引用的 ktlint 的格式化规则集 |
| naming         | 类名、变量命名相关的规范检查                   |
| performance    | 检查潜在的性能问题                        |
| potentail-bugs | 检查潜在的BUG                         |
| style          | 统一团队的代码风格，也包括一些由 Detekt 定义的格式化问题

表格引用自：[https://cloud.tencent.com/developer/article/1811025](https://cloud.tencent.com/developer/article/1811025)

更细节的规则说明，请参考：[官方文档-规则集说明](https://detekt.dev/docs/rules/comments)

### 自定义规则
接下来我们自定义一个检测`KAE`使用的规则，如下所示： 

```kotlin
//  入口
class CustomRuleSetProvider : RuleSetProvider {
    override val ruleSetId: String = "detekt-custom-rules"
    override fun instance(config: Config): RuleSet = RuleSet(
        ruleSetId,
        listOf(
            NoSyntheticImportRule(),
        )
    )
}

// 自定义规则
class NoSyntheticImportRule : Rule() {
    override val issue = Issue(
        "NoSyntheticImport",
        Severity.Maintainability,
        "Don’t import Kotlin Synthetics as it is already deprecated.",
        Debt.TWENTY_MINS
    )

    override fun visitImportDirective(importDirective: KtImportDirective) {
        val import = importDirective.importPath?.pathStr
        if (import?.contains("kotlinx.android.synthetic") == true) {
            report(
                CodeSmell(
                    issue,
                    Entity.from(importDirective),
                    "'$import' 不要使用kae，推荐使用viewbinding"
                )
            )
        }
    }
}
```

代码其实并不复杂，主要做了这么几件事：   
1. 添加`CustomRuleSetProvider`作为自定义规则的入口，并将`NoSyntheticImportRule`添加进去
2. 实现`NoSyntheticImportRule`类，主要包括`issue`与各种`visitXXX`方法
3. `issue`属性用于定义在控制台或任何其他输出格式上打印的`ID`、严重性和提示信息
4. `visitImportDirective`即通过访问者模式访问语法树的回调，当访问到`import`时会回调，我们在这里检测有没有添加`kotlinx.android.synthetic`，发现存在则报告异常

### 支持类型解析的自定义规则
上面的规则没有用到类型解析，也就是说不传入`classpath`也能使用，我们现在来看一个需要使用类型解析的自定义规则

比如我们需要在项目中禁止直接使用`android.widget.Toast.show`，而是使用我们统一封装的工具类，那么我们可以自定义如下规则：   

```kotlin
class AvoidToUseToastRule : Rule() {
    override val issue = Issue(
        "AvoidUseToastRule",
        Severity.Maintainability,
        "Don’t use android.widget.Toast.show",
        Debt.TWENTY_MINS
    )

    override fun visitReferenceExpression(expression: KtReferenceExpression) {
        super.visitReferenceExpression(expression)
        if (expression.text == "makeText") {
            // 通过bindingContext获取语义
            val referenceDescriptor = bindingContext.get(BindingContext.REFERENCE_TARGET, expression)
            val packageName = referenceDescriptor?.containingPackage()?.asString()
            val className = referenceDescriptor?.containingDeclaration?.name?.asString()
            if (packageName == "android.widget" && className == "Toast") {
                report(
                    CodeSmell(
                        issue, Entity.from(expression), "禁止直接使用Toast，建议使用xxxUtils"
                    )
                )
            }
        }
    }
}
```

可以看出，我们在`visitReferenceExpression`回调中检测表达式，我们不仅需要判断是否存在`Toast.makeTest`表达式，因为可能存在同名类，更需要判断`Toast`类的具体类型，而这就需要获取语义信息

我们这里通过`bindingContext`来获取表达式的语义，这里的`bindingContext`其实就是`Kotlin`编译器存储语义信息的表，详细的可以参阅：[K2 编译器是什么？世界第二高峰又是哪座？](https://juejin.cn/post/7143207967775522823#heading-8)

当我们获取了语义信息之后，就可以获取`Toast`的具体类型，就可以判断出这个`Toast`是不是`android.widget.Toast`，也就可以完成检测了

## `Github Action`集成`Detekt`检测
在完成了`DeteKt`接入与自定义规则之后，接下来就是每次提交代码时在`CI`上进行检测了

一些大的开源项目每次提交`PR`都会进行一系列的检测，我们也用`Github Action`来实现一个

我们在`.github/workflows`目录添加如下代码

```
name: Android CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  detekt-code-check:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'
        cache: gradle

    - name: Grant execute permission for gradlew
      run: chmod +x gradlew
    - name: DeteKt Code Check
      run: ./gradlew detektDebug
```

这样在每次提交`PR`的时候，就都会自动调用该`workflow`进行检测了，检测不通过则不允许合并，如下所示：  

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p18.webp)

点进去也可以看到详细的报错，具体是哪一行代码检测不通过，如图所示：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/july/p19.webp)

## 总结
本文主要介绍了`DeteKt`的接入与如何自定义规则，通过`IDE`集成，`CLI`命令行方式与`Gradle`插件方式接入，以及`CI`自动检测，可以保证代码规范，`IDE`提示，`CI`检测三者的统一，方便提前暴露问题，提高代码质量。

如果本文对你有所帮助，欢迎点赞~

### 示例代码
本文所有代码可见：[https://github.com/RicardoJiang/android-workflow](https://github.com/RicardoJiang/android-workflow)

### 参考资料
[https://detekt.dev/docs/intro](https://detekt.dev/docs/intro)           
[代码质量堪忧？用 detekt 呀，拿捏得死死的~](https://cloud.tencent.com/developer/article/1811025)
