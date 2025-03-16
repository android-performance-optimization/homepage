# 如何给 Kotlin 新增一个 target?
去年看了一本书：[《计算机系统要素-从零开始构建现代计算机》](https://android-performance-optimization.github.io/foundation/nand-to-tetris/recommend/), 这本书从与非门开始引导你一步步构建一个计算机，完成本书的所有项目后你将获得以下收获：

- 从与非门开始构造出一个 16 位的 Hack 计算机
- 在此计算机基础之上，开发出汇编编译器、堆栈式虚拟机
- 针对虚拟机设计出高级编程语言 Jack，同时开发出相应的编译器及语言标准库

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/july/p12.png)

现在我们有了一个自己实现的 Jack 编程语言，Jack 虚拟机以及 Jack 字节码。Jack 编程语言毕竟是一个 demo 语言，语法写起来比较麻烦，那么我们能否利用 Kotlin 的多平台特性, 给 Kotlin 新增一个 target，把 Kotlin 编译成 Jack 字节码呢？

![](https://raw.gitmirror.com/RicardoJiang/resource/refs/heads/main/2025/march/p4.png)

## Jack 语言与 Jack 字节码
### Jack 语法简单示例
在给 Kotlin 新增一个 target 之前，我们需要先简单了解一下 Jack 语言与 Jack 字节码。

Jack 是一种面向对象的弱类型语言，语法简单，Jack 语言执行总是从 Main 类的 main 函数开始执行，所以每个 Jack 程序至少要包含一个 Main 类，该类必须包含一个 main 函数。

我们来看一个简单的 Jack 语言示例：

```
class Main {

   function void main() {
       do Output.printInt(1 + (2 * 3));
       return;
   }

}
```

- 在 Main 类中定义了一个 main 函数，这是程序的入口, 其中 function 代表静态函数。
- main 函数中调用了 Output 类的 printInt 函数，Output 类属于 Jack 标准库，printInt 函数用于打印整数。
- 每个语句以分号结尾，return 语句用于结束函数。

### Jack 虚拟机
Jack 虚拟机是基于堆栈的（stack-based）：所有的操作都在堆栈上完成。它也是基于函数的（function-based）：一个完整的、应用 VM 语言编写的 VM 程序由若干个称函数（functions）的程序单元组成，这些函数使用VM 语言编写。该语言包含四种类型的命令：

- 算术命令: 在堆栈上执行算术和逻辑操作。
- 内存访问命令: 在堆栈和虚拟内存单元之间转移数据。
- 程序流程命令: 使条件分支操作和无条件分支操作变得容易。
- 函数调用命令: 调用函数并返回调用处（即函数调用指令的下一条指令地址）。

其中算术命令就是加减乘除等操作，函数调用命令就是调用函数，程序流程命令就是 if else 等操作，在下个示例详细介绍。内存访问命令用于在堆栈和虚拟内存单元之间转移数据，更为复杂一些。

内存访问命令使用命令 pop 和 push x 来表示，这里符号 x 代表在某个全局内存中的一个独立的存储单元。为了保留语义信息，VM 需要操纵 8 个独立的虚拟内存段，如下图所示：

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/july/p14.png)

接下来，我们看上面的 Jack 语言示例编译成 Jack 字节码后的结果：

```
function Main.main 0
    push constant 1
    push constant 2
    push constant 3
    call Math.multiply 2
    add
    call Output.printInt 1
    pop temp 0
    push constant 0
    return
```

- function Main.main 0: 定义了一个 Main 类的 main 函数，0 代表该函数内部局部变量数。
- push constant x: 将常量 x 压入堆栈。
- call Math.multiply 2: 调用 Math 类的 multiply 函数，2 代表该函数参数个数，从堆栈中弹出两个参数，将函数结果压入堆栈。
- add: 从堆栈中弹出两个整数，相加后将结果压入堆栈。
- call Output.printInt 1: 调用 Output 类的 printInt 函数，1 代表该函数参数个数，从堆栈中弹出一个参数，将函数结果压入堆栈。
- pop temp 0: Output.printInt 函数返回值不需要，所以将其弹出。
- jack 默认每个函数都需要返回一个值，所以最后需要 push constant 0 和 return。

### 程序流程命令示例
接下来我们看一个程序流程命令示例：

```
class Main {
   function int fibonacciRecursive(int n) {
          if (n < 2) {
             return n;
          } else {
            return Main.fibonacciRecursive(n-1) + Main.fibonacciRecursive(n-2);
           }
      }

      function void main() {
         var int result;
         let result = Main.fibonacciRecursive(4);
         do Output.printInt(result);
         return;
      }
}
```

与上面的示例类似，这里定义了一个 Main 类，其中包含了两个函数：一个是 fibonacciRecursive 函数，用于计算斐波那契数列；另一个是 main 函数，用于调用 fibonacciRecursive 函数并打印结果。

不同的是，这里通过 if else 语句实现了程序流程控制，并且通过 var 关键字定义了一个局部变量 result，通过 let 关键字给 result 赋值。

上面的代码是一个递归计算斐波那契数列的例子，我们来看一下这段代码编译成 Jack 字节码后的结果：

```
function Main.fibonacciRecursive 0
    push argument 0
    push constant 2
    lt
    not
    if-goto Main_1
    push argument 0
    return
    goto Main_0
label Main_1
    push argument 0
    push constant 1
    sub
    call Main.fibonacciRecursive 1
    push argument 0
    push constant 2
    sub
    call Main.fibonacciRecursive 1
    add
    return
label Main_0
function Main.main 1
    push constant 4
    call Main.fibonacciRecursive 1
    pop local 0
    push local 0
    call Output.printInt 1
    pop temp 0
    push constant 0
    return
```

- function Main.fibonacciRecursive 0: 定义了一个 Main 类的 fibonacciRecursive 函数，0 代表该函数内部局部变量数。
- push argument 0: 将参数 n 压入堆栈。
- push constant 2: 将常量 2 压入堆栈。
- lt: 从堆栈中弹出两个整数，比较大小，将结果压入堆栈。
- not: 从堆栈中弹出一个整数，取反，将结果压入堆栈。这里取反是为了方便 if-goto 判断。
- if-goto Main_1: 从堆栈中弹出一个整数，如果为真则跳转到 Main_1 标签。
- ...

## Kotlin 编译成 Jack 字节码
之前介绍过 Kotlin/Jvm 的编译流程：[Kotlin/JVM 编译流程浅析](https://android-performance-optimization.github.io/foundation/kotlin-compiler-learn/kotlin-compiler-source/)

![](https://raw.gitmirror.com/RicardoJiang/resource/refs/heads/main/2024/december/p8.png)

因此当我们要给 Kotlin 新增一个 target 时，得益于 Kotlin 良好的分层架构，编译器前端部分基本不需要改动，最主要的工作在于把 IR 转化为 Jack 字节码，具体来说需要实现以下几个步骤：

- 添加 kotlin/Jack 标准库：如果需要使用一些平台独有的函数，需要实现相应的标准库，在这里我们只需要实现 Output 类，因此可以直接使用 Kotlin/Js 的标准库。
- 添加 cli-jack 模块：实现 Jack target 的入口类，负责解析命令行参数，调用 Kotlin 编译器生成 IR，然后调用 backend-jack 模块将 IR 转化为 Jack 字节码。
- 添加 backend-jack 模块：实现 IR 转化为 Jack 字节码的逻辑。

### 添加 cli-jack 模块
当我们运行 kotlinc 编译代码时，实际上会运行到`org.jetbrains.kotlin.preloading.Preloader`类，然后根据参数的不同，调用到不同编译 target 的入口类，因此我们需要添加一个 cli-jack 模块，实现 Jack 编译 target 的入口类，串起编译的整体逻辑。

```kotlin
class K2JackCompiler() : CLICompiler<K2JSCompilerArguments>() {
    class K2JSCompilerPerformanceManager : CommonCompilerPerformanceManager("Kotlin to Jack Compiler")

    override val defaultPerformanceManager: CommonCompilerPerformanceManager = K2JSCompilerPerformanceManager()
    override fun createMetadataVersion(versionArray: IntArray): BinaryVersion {
        return KlibMetadataVersion(*versionArray)
    }

    override fun createArguments(): K2JSCompilerArguments {
        return K2JSCompilerArguments()
    }

    override fun executableScriptFileName(): String {
        return "kotlinc-jack"
    }

    override fun MutableList<String>.addPlatformOptions(arguments: K2JSCompilerArguments) {
    }

    override fun doExecute(
        arguments: K2JSCompilerArguments,
        configuration: CompilerConfiguration,
        rootDisposable: Disposable,
        paths: KotlinPaths?,
    ): ExitCode {
        // 1. 解析命令行参数
        val outputDirPath = arguments.outputDir ?: return COMPILATION_ERROR
        val outputName = arguments.moduleName ?: return COMPILATION_ERROR

        // 2. 生成 fir
        val firOutput = compileModulesToAnalyzedFirWithLightTree(
            moduleStructure = moduleStructure,
            groupedSources = groupedSources,
            // TODO: Only pass groupedSources, because
            //  we will need to have them separated again
            //  in createSessionsForLegacyMppProject anyway
            ktSourceFiles = groupedSources.commonSources + groupedSources.platformSources,
            libraries = libraries,
            friendLibraries = friendLibraries,
            diagnosticsReporter = diagnosticsReporter,
            incrementalDataProvider = configuration[JSConfigurationKeys.INCREMENTAL_DATA_PROVIDER],
            lookupTracker = lookupTracker,
            useWasmPlatform = arguments.wasm,
        )

        // 3. fir 转化为 ir
        val fir2IrActualizedResult = transformFirToIr(moduleStructure, firOutput.output, diagnosticsReporter)

        // 4. ir 转化为 Jack 字节码
        IrModuleToJackTransformer().generateCode(fir2IrActualizedResult.irModuleFragment, outputDirPath, outputName)
        return OK
    }
}
```

cli-jack 模块主要实现了解析命令行参数，串起编译的整体逻辑，其中参数解析，生成 fir，fir 转化为 ir 的逻辑基本可以复用，我们这里复用了 Kotlin/Js 的逻辑，因此我们只需要实现 ir 转化为 Jack 字节码的逻辑即可。

### 添加 backend-jack 模块
backend-jack 模块主要实现了 ir 转化为 Jack 字节码的逻辑，这里我们需要实现一个 IrModuleToJackTransformer 类，负责将 ir 转化为 Jack 字节码。

```kotlin
class IrModuleToJackTransformer {
    fun generateCode(irModule: IrModuleFragment, outputDirPath: String, outputName: String) {
        irModule.files.forEach { file ->
            val context = JackGenerationContext(outputDirPath, outputName)
            file.accept(
                IrFileToJackTransformer(),
                data = context
            )
        }
    }
}

class IrFileToJackTransformer : BaseIrElementToJackTransformer {

    override fun visitFile(declaration: IrFile, context: JackGenerationContext) {
        super.visitFile(declaration, context)
        declaration.declarations.forEach {
            it.accept(IrDeclarationToJackTransformer(), context)
        }
    }
}

// ...
```

在 backend-jack 模块中，我们主要通过访问者模式实现了 ir 转化为 Jack 字节码的逻辑，通过访问 ir 的不同节点，生成对应的 Jack 字节码。具体的代码就不在这里展示了，感兴趣的同学可以查看源码：[https://github.com/RicardoJiang/kotlin/tree/jack](https://github.com/RicardoJiang/kotlin/tree/jack)

在这里我们只实现了基本的变量声明与赋值，函数声明与调用，条件分支，while 循环支持等逻辑，更复杂的对象创建与销毁、字符串支持、数组支持等可以根据 Jack 字节码规范自行实现。

### 生成与运行 Jack 字节码
当我们完成 backend-jack 模块的开发后，我们就可以直接运行`org.jetbrains.kotlin.preloading.Preloader`类来编译 Kotlin 代码生成 Jack 字节码了。

```
package org.jetbrains.kotlin.preloading;

@SuppressWarnings("UseOfSystemOutOrSystemErr")
public class Preloader {
    public static void main(String[] args) throws Exception {
        
        try {
            // 写死 kotlin-compiler.jar 的路径，标准库路径，输出路径，输出名字，输入文件路径等参数
            String[] testArgs = {"-cp",  "./dist/kotlinc/lib/kotlin-compiler.jar", "org.jetbrains.kotlin.cli.jack.K2JackCompiler",
                    "-libraries","./dist/kotlinc/lib/kotlin-stdlib-js.klib",
                    "-ir-output-dir","/Users/jiangjunxiang/AndroidProject/leo/kotlin/compilerTestData","-ir-output-name","FibRecursive",
                    "/Users/jiangjunxiang/AndroidProject/leo/kotlin/compilerTestData/FibRecursive.kt"
            };
            run(testArgs);
        }
        catch (PreloaderException e) {
            System.err.println("error: " + e.toString());
        }
    }
}
```

运行上面的代码后，我们就可以在指定的输出路径下看到生成的 Jack 字节码文件了。生成字节码后，可以在在线平台上运行 Jack 字节码，以验证其正确性：[https://nand2tetris.github.io/web-ide/vm](https://nand2tetris.github.io/web-ide/vm)

![](https://raw.gitmirror.com/RicardoJiang/resource/refs/heads/main/2025/march/p3.png)

## 总结
本文主要介绍了 Jack 语言与 Jack 字节码，以及如何给 Kotlin 新增一个 target，将 Kotlin 编译成 Jack 字节码。通过这个例子，我们可以看到得益于 Kotlin 良好的分层架构，给 Kotlin 新增一个 target 并不是一件困难的事情，只需要实现 backend-jack 模块，串起编译的整体逻辑即可。