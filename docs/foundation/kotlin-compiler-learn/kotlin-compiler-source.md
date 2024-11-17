# 破解 Kotlin 编译器

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