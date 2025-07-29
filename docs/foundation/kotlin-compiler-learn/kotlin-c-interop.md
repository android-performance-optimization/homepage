# Kotlin 调用 C 语言是如何实现的?
Kotlin/Native 通过 C-interop 机制实现与 C 语言的互操作性，允许 Kotlin 代码直接调用 C 函数和使用 C 数据结构。C-interop 通过解析 C 语言头文件自动生成相应的 Kotlin 绑定代码，为 Kotlin 与 C 库的交互提供了无缝的桥梁。

我们就一起来看下，Kotlin 调用 C 语言具体是如何实现的。

## 接口定义
```
headers = hello.h

# (For cinterop tool) Path to search for static libraries.
# Must be relative to the project root.
libraryPaths = src/nativeInterop/cinterop

# (For final linker) Options to be embedded in the klib.
# Must be relative to the project root.
linkerOpts = -Lsrc/nativeInterop/cinterop

# The static library file to be included.
staticLibraries = libhello.a

# (For cinterop tool) Path to search for header files.
# Must be relative to the project root.
compilerOpts = -Isrc/nativeInterop/cinterop
```

如上所示，`.def` 文件定义了 C 接口，包括头文件、编译选项、链接选项等信息。C-interop 工具将解析这个文件，并生成相应的 Kotlin 代码和 KLIB 文件。

## 绑定与桥接生成
```
./kotlin-native/dist/bin/run_konan cinterop -def ./compilerTestData/cinterop/hello.def -o compilerTestData/cinterop/Hello -J"-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:50015"
```

当我们执行 C-interop 命令时，实际上是调用了以上命令，这个命令会执行以下操作：
### 参数解析
执行以上命令行时，会执行到 mainImpl 函数，该函数负责解析命令行参数，并根据参数调用相应的 C-interop 工具。

```
private fun mainImpl(args: Array<String>, runFromDaemon: Boolean, konancMain: (Array<String>) -> Unit) {
    val utilityName = args[0]
    val utilityArgs = args.drop(1).toTypedArray()
    when (utilityName) {
        "cinterop" -> {
            val konancArgs = invokeInterop("native", utilityArgs, runFromDaemon)
            konancArgs?.let { konancMain(it) }
        }
        // ...
    }
}
```

### 调用核心生成器
在解析完参数后，C-interop 工具会调用核心生成器来处理 C 库的绑定生成。这个过程主要由 `processCLib` 函数完成。

`processCLib` 的主要职责是：获取 C 库的完整定义，调用 LLVM 解析其头文件，生成所有必要的 Kotlin 和 C 的“胶水代码”（stubs），并将其编译打包，为最终生成 KLIB 做好准备。

#### 获取 C 库定义
在 `processCLib` 函数中，首先会获取 C 库的定义, 这一步通过调用 LLVM 解析 `.def` 中指定的头文件来完成的。

```
// nativeIndex 是一个抽象类，它描述了来自C头文件的定义的IR
val (nativeIndex, compilation) = buildNativeIndexImpl(library, verbose, allowPrecompiledHeaders = nativeLibsDir != null)
```

#### 构建 Kotlin IR 存根
这一步的主要任务是从 C AST 中提取所有声明信息，并将 C 声明转换为 Kotlin IR 存根。

```kotlin
class StubIrBuilder(private val context: StubIrContext) {
    fun build(): StubIrBuilderResult {
        // ...
        nativeIndex.functions.filter { it.name !in excludedFunctions }.forEach { generateStubsForFunction(it) }
        // ...
        return StubIrBuilderResult(
                stubs,
                buildingContext.declarationMapper,
                buildingContext.bridgeComponentsBuilder.build(),
                buildingContext.wrapperComponentsBuilder.build()
        )
    }    
}

// 生成类型映射和函数签名
fun mirror(declarationMapper: DeclarationMapper, type: Type): TypeMirror = when (type) {
    is PrimitiveType -> mirrorPrimitiveType(type, declarationMapper)
    is RecordType -> byRefTypeMirror(declarationMapper.getKotlinClassForPointed(type.decl).type)
    is EnumType -> // ...
    is PointerType -> // ...
    is ArrayType -> // ...
    is FunctionType -> byRefTypeMirror(KotlinTypes.cFunction.typeWith(getKotlinFunctionType(declarationMapper, type)))
    is Typedef -> // ...
    is ObjCPointer -> objCPointerMirror(declarationMapper, type)
    else -> TODO(type.toString())
}
```

这一步会处理基础类型（如整数、浮点数等）和复杂类型（如结构体、枚举等）的映射转换。它会生成相应的 Kotlin 类型，以便在 Kotlin 代码中使用 C 数据结构

这一步生成的内容实际上就是 CInterop 生成的 knm(kotlin native metadata) 文件，生成的内容就是函数声明，如下所示：

```
package hello

@kotlinx.cinterop.internal.CCall 
@kotlinx.cinterop.ExperimentalForeignApi 
public external fun hello_from_c(timestamp: kotlin.Long): kotlin.Unit { /* compiled code */ }
```

#### 构建桥接代码
```
class StubIrDriver(
        private val context: StubIrContext,
        private val options: DriverOptions
) {
    fun run(): Result {
        // 生成 Kotlin IR 存根
        val builderResult = StubIrBuilder(context).build()
        // 构建桥接代码
        val bridgeBuilderResult = StubIrBridgeBuilder(context, builderResult).build()
        // 生成 C 存根文件
        outCFile.bufferedWriter().use {
            emitCFile(context, it, entryPoint, bridgeBuilderResult.nativeBridges)
        }
        // ...
    }
}
```

这一步的主要作用是生成 Kotlin 和 C 之间的桥接代码并写入 C 存根文件。它会创建函数调用包装器，生成属性访问器，并处理类型转换。

生成的内容如下所示，可以看到生成了`hello_from_c` 函数的调用包装器，后续 Kotlin 代码调用这个函数时，会通过这个包装器来调用 C 函数。

```c
// __attribute__((always_inline)): 强制内联优化，减少函数调用开销
__attribute__((always_inline))
// 函数名 hello_hello_from_c_wrapper0 由 Kotlin/Native 自动生成
// 参数 long long p0 对应 Kotlin 的 Long 类型
void hello_hello_from_c_wrapper0(long long p0) {
	hello_from_c(p0);
}

// 声明函数指针变量，用于存储包装器函数的地址
// __asm("knifunptr_hello0_hello_from_c") 指定汇编符号名，避免命名冲突
const void* knifunptr_hello0_hello_from_c __asm("knifunptr_hello0_hello_from_c");
// 将包装器函数的地址赋值给函数指针变量
// 这样 Kotlin 代码就可以通过这个函数指针安全地调用 C 函数
const void* knifunptr_hello0_hello_from_c = (const void*)&hello_hello_from_c_wrapper0;
```

#### 原生代码编译
上面生成的 C 桥接代码，需要通过 LLVM 编译为 bc 文件。

```
when (flavor) {
    KotlinPlatform.NATIVE -> {
        // 编译为 LLVM 位码 (.bc) 文件
        val outLib = File(nativeLibsDir, "$libName.bc")
        val compilerCmd = arrayOf(compiler, *compilerArgs,
            "-emit-llvm", "-x", library.language.clangLanguageName, 
            "-c", "-", "-o", outLib.absolutePath)
        runCmd(compilerCmd, verbose, redirectInputFile = File(outCFile.absolutePath))
    }
}
```

#### 生成 KLIB
最后一步是将生成的 Kotlin IR 存根和桥接代码序列化并写入 KLIB 文件。这一步会将库的元数据、IR Stub、manifest 文件等所有内容，按照 .klib 文件格式的标准目录结构，写入到由 -o 参数指定的输出目录中。

```kotlin
when (stubIrOutput) {
    is StubIrDriver.Result.Metadata -> {
        createInteropLibrary(
            metadata = stubIrOutput.metadata,           // Kotlin 元数据
            nativeBitcodeFiles = compiledFiles + nativeOutputPath,  // 原生位码
            target = tool.target,                       // 目标平台
            moduleName = moduleName,                    // 模块名
            outputPath = outputPath,                    // 输出路径
            manifest = def.manifestAddendProperties,    // 清单属性
            dependencies = stdlibDependency + imports.requiredLibraries,
            nopack = nopack,                           // 是否打包
            shortName = cinteropArguments.shortModuleName,
            staticLibraries = resolveLibraries(staticLibraries, libraryPaths)
        )
    }
}
```

## 编译器处理 @CCall 函数调用
上面生成的 KLIB 文件，包含了 Kotlin IR 存根和桥接代码。接下来，Kotlin 编译器会处理这个 KLIB 文件，将其与 Kotlin 代码进行链接。通过以下命令行执行 Kotlin 编译器：

```
./kotlin-native/dist/bin/run_konan konanc ./compilerTestData/cinterop/Main.kt -library ./compilerTestData/cinterop/Hello.klib -o ./compilerTestData/cinterop/Hello -J"-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:50015"
``` 

同时我们可以看到，上面生成的 knm 中的函数为`hello_from_c`，而生成的桥接代码中的函数是`knifunptr_hello0_hello_from_c`，因此 Kotlin 编译器还需要识别`@kotlinx.cinterop.internal.CCall`函数，并修改调用代码，将其替换为桥接函数。

```kotlin
// 处理 C 函数调用
if (function.annotations.hasAnnotation(RuntimeNames.cCall)) {
    return generateCCall(expression)、
}

generateCCall(expression: IrCall, builder: IrBuilderWithScope, isInvoke: Boolean,
                                       foreignExceptionMode: ForeignExceptionMode.Mode = ForeignExceptionMode.default): IrExpression {
    // ...
    if (isInvoke) {
        callBuilder.cBridgeBodyLines.add(0, "$targetFunctionVariable = ${targetPtrParameter!!};")
    } else {
        val cCallSymbolName = callee.getAnnotationArgumentValue<String>(RuntimeNames.cCall, "id")!!
        this.addC(listOf("extern const $targetFunctionVariable __asm(\"$cCallSymbolName\");")) // Exported from cinterop stubs.
    }
    return result
}
```

## 运行时处理
我们知道，Kotlin/Native 使用自动垃圾回收，而 C 语言使用手动内存管理，因此在 Kotlin 调用 C 函数时，可能导致了以下关键问题：

- 生命周期不匹配：Kotlin 对象可能被 GC 回收，而 C 代码仍持有引用
- 内存泄漏风险：C 代码分配的内存需要手动释放
- 数据所有权不明确：谁负责释放内存？

Kotlin/Native 通过运行时库提供了 C-interop 的支持，处理这些问题。具体原理在介绍 Kotlin/Native 内存管理机制时再介绍。

## 总结
Kotlin/Native 与 C 语言的互操作性是其核心特性之一，它通过 `cinterop` 工具链实现。该过程首先通过一个 `.def` 配置文件来声明需要交互的 C 库头文件和链接信息。

`cinterop` 工具会解析这些头文件，并执行以下关键操作：

1.  **生成 Kotlin IR 存根**：创建 C 函数和类型的 Kotlin 声明，并将其存储为 `.klib` 中的元数据。这使得 Kotlin 编译器能够识别这些外部符号。
2.  **生成 C 桥接代码**：创建 C “胶水”函数（wrapper），用于安全地调用原始 C 函数。这些桥接代码随后被编译成原生 LLVM 位码。

最终，Kotlin 元数据和原生位码被一同打包进一个 `.klib` 文件。当编译器在用户代码中遇到对 C 函数的调用时（通过 `@CCall` 注解识别），它会链接到 `.klib` 中对应的桥接函数，从而完成整个调用链路。这个精巧的设计使得开发者可以在 Kotlin 中以类型安全的方式无缝调用底层 C 代码。