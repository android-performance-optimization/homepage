# Kotlin 编译器浅析

## 一般的编译流程
编译器一般由以下几个部分组成：

- 词法分析器（Lexical Analyzer）：也称为扫描器（Scanner），它负责将源代码转化为一系列的记号（tokens）。每一个记号代表源代码中的最小有意义单元，如关键字、标识符、操作符、分隔符等。
- 语法分析器（Syntax Analyzer）：也称为解析器（Parser），它根据词法分析器生成的记号，按照一定的语法规则生成语法树（或称为解析树）。语法树表达了输入程序的语法结构。
- 语义分析器（Semantic Analyzer）：在这一步，编译器检查语法树中是否存在语义错误，比如类型检查、作用域检查等。语义分析可能会生成某种形式的中间表示（Intermediate Representation, IR）。
- 中间代码生成器（Intermediate Code Generator）：将高层的源代码转换为较低层的、机器无关的中间代码。这种中间代码便于优化和翻译成机器码。
- 代码优化器（Code Optimizer）：对中间代码进行优化，目的是提高程序的执行效率或减少代码的占用空间。优化方法包括移除无用代码、进行常量传播、循环优化等。
- 目标代码生成器（Code Generator）：将优化后的中间代码转换为目标机器代码或汇编代码（assembly code）。

## Kotlin 编译器如何调试?
```
java_version="$(findJavaVersion)"
if [[ $java_version -ge 9 ]]; then
    # Workaround the illegal reflective access warning from ReflectionUtil to ResourceBundle.setParent, see IDEA-248785.
    java_args=("${java_args[@]}" "--add-opens" "java.base/java.util=ALL-UNNAMED")
fi

if [ -n "$KOTLIN_RUNNER" ]; then
    java_args=("${java_args[@]}" "-Dkotlin.home=${KOTLIN_HOME}")
    kotlin_app=("${KOTLIN_HOME}/lib/kotlin-runner.jar" "org.jetbrains.kotlin.runner.Main")
else
    [ -n "$KOTLIN_COMPILER" ] || KOTLIN_COMPILER=org.jetbrains.kotlin.cli.jvm.K2JVMCompiler

    if [[ $java_version < 13 ]]; then
        java_args=("${java_args[@]}" "-noverify")
    fi

    declare additional_classpath=""
    if [ -n "$KOTLIN_TOOL" ]; then
        additional_classpath=":${KOTLIN_HOME}/lib/${KOTLIN_TOOL}"
    fi

    kotlin_app=("${KOTLIN_HOME}/lib/kotlin-preloader.jar" "org.jetbrains.kotlin.preloading.Preloader" "-cp" "${KOTLIN_HOME}/lib/kotlin-compiler.jar${additional_classpath}" $KOTLIN_COMPILER)
fi

"${JAVACMD:=java}" $JAVA_OPTS "${java_args[@]}" -cp "${kotlin_app[@]}" "${kotlin_args[@]}"
```

通过查看 kotlinc 源码可知，`org.jetbrains.kotlin.preloading.Preloader` 类是编译器的预加载器，它会加载编译器的主类 `org.jetbrains.kotlin.cli.jvm.K2JVMCompiler`。因此我们修改`Preloader`类，运行它的`main`方法，即可启动与调试编译器，示例如下所示:

```java
String[] testArgs = {"-cp",  "./dist/kotlinc/lib/kotlin-compiler.jar", "org.jetbrains.kotlin.cli.jvm.K2JVMCompiler",
                    "./compilerTestData/Test.kt", "-include-runtime",
                    "-d", "./compilerTestData/Test.jar"
            };
            run(testArgs);
```

## 词法分析
这一步的输入是源代码，输出是一系列的记号（tokens），每个记号代表源代码中的词法单元，如关键字、标识符、操作符、分隔符，注释等。

转换的过程是通过扫描源代码，根据 Kotlin 的词法规则，将源代码分割成一个个的词法单元。这个过程是由词法分析器（Lexical Analyzer）完成的。

由于词法分析器已经是一个比较成熟的技术，因此我们可以直接使用现成的工具来生成词法分析器。比如，Kotlin 就是使用了 [JFlex](https://jflex.de/) 来生成词法分析器，通过编写 JFlex 规范文件 [/Kotlin.flex](https://github.com/JetBrains/kotlin/blob/5f70647879916e79704ac3dd4a3c60ff27554503/compiler/psi/src/org/jetbrains/kotlin/lexer/Kotlin.flex#L4)，然后使用 JFlex 工具即可生成词法分析器。

> JFlex 是一个词法分析器生成器（也称为扫描器生成器），使用 Java 编写。

> 一个词法分析器生成器的输入是一个包含一组正则表达式及其相应动作的规范。它生成一个程序（词法分析器），该程序读取输入，将输入与规范文件中的正则表达式进行匹配，如果某个正则表达式匹配成功，则执行相应的动作。词法分析器通常是编译器的第一个前端步骤，用于匹配关键字、注释、操作符等，并为解析器生成输入的标记流。

> JFlex 的词法分析器基于确定性有限自动机（DFA）。它们运行快速，没有昂贵的回溯操作。

> JFlex 设计用于与 Scott Hudson 的 LALR 解析器生成器 CUP 共同使用，以及用于 Bob Jamison 修改自 Berkeley Yacc 的 Java 版本 BYacc/J。它也可以与其他解析器生成器（如 ANTLR）一起使用，或者作为独立工具使用。

// TODO 字元化画图

## 语法分析
这一步的输入是 Tokens, 根据 Kotlin 的语法规则处理 Tokens，输出 AST(abstrace syntax tree)。

Kotlin 的语法规则可见：[https://kotlinlang.org/docs/reference/grammar.html](https://kotlinlang.org/docs/reference/grammar.html)。以下是 class 的定义，当遍历 tokens 列表时，如果碰到了 class 或者 interface 关键字，后续的 tokens 则按类或者 interface 来解析，其他语法规则也是如此，如果碰到不符合语法规则的语句则会报错。

```
classDeclaration (used by declaration)
  : modifiers? ('class' | ('fun'? 'interface'))
    simpleIdentifier typeParameters?
    primaryConstructor?
    (':' delegationSpecifiers)?
    typeConstraints?
    (classBody | enumClassBody)?
  ;
```

K1 与 K2 在语法分析阶段的输出略有不同, K2 在语法分析阶段会返回一个更加轻量级的语法树，构建速度更快，但后续无法创建 PSI。

### K2

```
  /**
   * 返回解析的结果。在调用此方法之前，所有的标记必须已经完成或被丢弃。
   *
   * @return 构建的 AST 树
   */
  @NotNull
  ASTNode getTreeBuilt();

  /**
   * 与 getTreeBuilt() 相同，但返回一个轻量级的树，构建速度更快，产生的垃圾更少，但无法创建一个 PSI（程序结构接口）。
   *
   * @return 轻量级的语法树
   */
  @NotNull
  FlyweightCapableTreeStructure<LighterASTNode> getLightTree();
```

相比 ASTNode，LighterASTNode 更加轻量化，接口中只有几个方法。

```
public interface LighterASTNode {
  LighterASTNode[] EMPTY_ARRAY = new LighterASTNode[0];

  IElementType getTokenType();

  int getStartOffset();

  int getEndOffset();

  default int getTextLength() {
    return getEndOffset() - getStartOffset();
  }
}
```

### K1
```kotlin
package compilerTestData

fun sample(a: Int, b: Int) {
    val c = a + b
    println("result:" + c)
}
```

![](https://raw.gitmirror.com/RicardoJiang/resource/refs/heads/main/2024/december/p5.png)

#### PSI

```
    override fun createElement(astNode: ASTNode): PsiElement {
        val elementType = astNode.elementType

        return when (elementType) {
            is KtStubElementType<*, *> -> elementType.createPsiFromAst(astNode)
            KtNodeTypes.TYPE_CODE_FRAGMENT, KtNodeTypes.EXPRESSION_CODE_FRAGMENT, KtNodeTypes.BLOCK_CODE_FRAGMENT -> ASTWrapperPsiElement(
                astNode
            )
            is KDocElementType -> elementType.createPsi(astNode)
            KDocTokens.MARKDOWN_LINK -> KDocLink(astNode)
            else -> (elementType as KtNodeType).createPsi(astNode)
        }
    }
```

在 IntelliJ Platform 中，解析文件是一个两步走的过程：
第一步，构建抽象语法树（AST），定义程序的结构。AST 节点由 IDE 内部创建，由 ASTNode 类的实例表示。每个 AST 节点都有一个关联的元素类型（IElementType 实例），这些元素类型由语言插件定义。文件的 AST 树顶层节点需要有一个特殊的元素类型，该类型继承自 IFileElementType 类。

ST 节点与底层文档中的文本范围有直接的映射关系。AST 的最底层节点匹配词法分析器返回的单个词法单元（token），而更高层的节点则匹配多个词法单元的片段。对 AST 树节点执行的操作（如插入、删除、重新排序节点等）会立即反映为底层文档文本的变化。

第二步，在 AST 之上构建程序结构接口（PSI）树，添加语义和用于操作特定语言结构的方法。PSI 树的节点由实现 PsiElement 接口的类表示，这些类由语言插件在 ParserDefinition.createElement() 方法中创建。文件的 PSI 树顶层节点需要实现 PsiFile 接口，并在 ParserDefinition.createFile() 方法中创建。

看起来 AST 和 PSI 都是代表语法树，那么为什么要再多一层呢？

实际上 PSI 在 IDEA 语法分析，代码补全等场景中大量使用，同时 IDEA 也支持自定义语言的支持，详情可见：[Custom Language Support](https://plugins.jetbrains.com/docs/intellij/custom-language-support.html)

Kotlin 在开发之初，应该是为了复用 IDEA 生态而使用了 PSI。但是 IDE 场景与编译器场景其实有些不同，IDE 会有频繁更新代码，快速导航等应用场景，因此 PSI 需要支持缓存，懒加载，
增量更新，导航等功能，而这些功能自然有一定成本，而在编译过程中实际并不需要，自然也会影响编译性能。

## 语义分析
### K1
#### BindingContext
BindingContext - 存储分析过程中收集的所有信息：
声明到描述符的映射
表达式类型信息
诊断信息
解析的导入
作用域信息

SlicedMap 的设计允许编译器：
1. 高效地存储和访问编译过程中的各种映射关系
通过不同的 Slice 类型支持不同的数据访问模式
3. 提供类型安全的数据访问
支持灵活的重写策略
支持值的延迟计算

#### ModuleDescriptor - 模块级别的信息：
1. 结构信息：
包层次结构
依赖关系图
模块组织结构
类型系统：
内建类型支持
平台特定类型
类型别名
多平台支持：
平台特定代码
expect/actual 声明
跨平台兼容性
访问控制：
模块间可见性
包级别访问
内部成员控制
扩展性：
模块能力系统
自定义特性支持
平台扩展
这种设计使得 ModuleDescriptor 能够：
支持模块化开发
提供类型安全
管理依赖关系
控制代码访问
支持多平台项目


## IR 生成
