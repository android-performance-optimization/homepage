# Kotlin/Native 编译流程浅析
之前我们介绍了 Kotlin/JVM 的编译流程，今天我们来看看 Kotlin/Native 的编译流程。本文基于 Kotlin 2.2 代码分析。

## Kotlin/Native 编译器如何调试
- 构建 Kotlin/Native 编译器: ./gradlew :kotlin-native:dist
- 运行编译器: ./kotlin-native/dist/bin/run_konan  konanc ./compilerTestData/Hello.kt -o ./compilerTestData/Hello  -J"-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:50015"
- 上面命令中的-agentlib:jdwp=... 是 JVM 的调试参数，用于开启远程调试支持，运行以上命令后，可以使用 IDEA 附加调试到 50015 端口进行调试。

## 入口阶段
当我们通过命令行运行 Kotlin/Native 编译器时，首先会调用到 `K2Native.main()` 方法中。

```
K2Native.main() 
    ↓
K2Native.doExecute()
    ↓ 
KonanDriver.run()
    ↓ 
KonanDriver.splitOntoTwoStages()
```

值得注意的是，在 splitOntoTwoStages 方法中，Kotlin/Native 的编译流程被分为两个阶段：

- 第一阶段：源代码文件由编译器前端编译成中间 KLib
- 第二阶段：中间 KLib 由 K2/Native 后端编译为二进制文件

## 生成中间 KLib
源代码文件由编译器前端编译成中间 KLib，也分为 K1 与 K2 两种方式，这里只介绍 K2 的方式。

```
val frontendOutput = engine.runFirFrontend(environment)     // 前端编译生成 FIR
val fir2IrOutput = engine.runFir2Ir(frontendOutput)         // FIR → IR
val loweredIr = engine.runPreSerializationLowerings(...)    // IR 降级
engine.runFir2IrSerializer(FirSerializerInput(loweredIr))   // 序列化
serializerOutput?.let { engine.writeKlib(it) }              // 写入本地 KLib 文件
```

在 K2 的编译流程中，前端编译生成 FIR（前端中间表示），然后将 FIR 转换为 IR（中间表示），接着进行 IR 的降级处理，最后序列化为 KLib。

其中生成 FIR 与 FIR 转 IR 的过程与 Kotlin/JVM 是复用的，可以参照[Kotlin/JVM 编译流程浅析](https://android-performance-optimization.github.io/foundation/kotlin-compiler-learn/kotlin-compiler-source/)，这里就不再赘述。生成的 IR 经过降级后，序列化为如下格式的 klib 文件。

```
klib
├── ir                              # IR 相关文件
│   ├── bodies.knb
│   ├── debugInfo.knd
│   ├── files.knf
│   ├── irDeclarations.knd
│   ├── signatures.knt
│   ├── strings.knt
│   └── types.knt
├── linkdata                        # 链接时元数据
│   ├── module
│   ├── package_com
│   │   └── 0_com.knm
│   ├── package_com.jarvis
│   │   └── 0_jarvis.knm
│   ├── package_com.jarvis.kmm
│   │   └── 0_kmm.knm
│   ├── package_com.jarvis.kmm.infra
│   │   └── 0_infra.knm
│   └── root_package
│       └── 0_.knm
├── manifest                        # 模块元数据
├── resources                       # 资源文件
└── targets                         # 目标平台相关文件
    └── ios_arm64
        ├── included
        ├── kotlin
        └── native
```

## 生成二进制文件(produceBinary)
在第二阶段，Kotlin/Native 的编译器将中间 KLib 与依赖的 KLib 进行编译链接，最终生成二进制文件。

```kotlin
private fun produceBinary(engine: PhaseEngine<PhaseContext>, config: KonanConfig, environment: KotlinCoreEnvironment) {
        val frontendOutput = performanceManager.tryMeasurePhaseTime(PhaseType.Analysis) { engine.runFrontend(config, environment) } ?: return

        val psiToIrOutput = performanceManager.tryMeasurePhaseTime(PhaseType.TranslationToIr) { engine.runPsiToIr(frontendOutput, isProducingLibrary = false) }
        require(psiToIrOutput is PsiToIrOutput.ForBackend)

        val backendContext = createBackendContext(config, frontendOutput, psiToIrOutput)
        engine.runBackend(backendContext, psiToIrOutput.irModule, performanceManager)
    }
```

### 前端分析
从 produceBinary 方法可以看到，这里要再运行一遍`runFrontend`。这一步的输入是上一阶段生成的 KLib 文件，与依赖的 KLib 文件。输出是前端分析结果，具体包括

- moduleDescriptor: 当前模块的完整描述, 包含所有依赖模块的引用
- bindingContext: 由于没有源码，这里的 bindingContext 为空
- environment： 由于没有源码，这里的 psi 列表也为空

这个阶段因为源码列表为空，PSI 和 BindingContext 都是空的，但模块级别的元数据是完整的，这是后续 IR 反序列化的基础。

```kotlin
internal val FrontendPhase = createSimpleNamedCompilerPhase(
        "Frontend",
        outputIfNotEnabled = { _, _, _, _ -> FrontendPhaseOutput.ShouldNotGenerateCode }
) { context: FrontendContext, input: KotlinCoreEnvironment ->
    // ...
}
```

### IR 反序列化
produceBinary 中的 runPsiToIr 与普通的 PSI 到 IR 转换不同，主要是从 klib 反序列化 IR 模块。

```kotlin
internal val PsiToIrPhase = createSimpleNamedCompilerPhase<PsiToIrContext, PsiToIrInput, PsiToIrOutput>(
        "PsiToIr",
        postactions = getDefaultIrActions(),
        outputIfNotEnabled = { _, _, _, _ -> error("PsiToIr phase cannot be disabled") }
) { context, input ->
    context.psiToIr(input, useLinkerWhenProducingLibrary = false)
}
```

### IR 降级阶段（IR Lowering Phase）
lowering 是指将高级的 Kotlin 代码转换成更加底层的、容易被后续编译阶段或目标平台（如 JVM 字节码、JavaScript、原生代码等）理解并处理的代码的过程。这个过程通常包括将更抽象、高级的语言特性分解成更基础的元素

1. **验证和预处理**: validateIrBeforeLowering
2. **合成访问器生成**: SyntheticAccessors  
3. **内联优化**: 
   - inlineAllFunctionsPhase
   - 内联私有函数
4. **平台特定处理**: specialObjCValidationPhase  
5. **常量计算**: constEvaluationPhase
6. **后处理降级**: getLoweringsAfterInlining()
7. **最终验证**: validateIrAfterLowering

### IR 转化为 LLVM IR(CodegenPhase)
Kotlin IR 转换为 LLVM IR 的过程也是通过访问者模式遍历 IrFile 及其子节点，然后调用 LLVM 相关的 API 进行转换。

如下是一个简单的 Kotlin 函数转换为 LLVM IR 的示例：

```
fun add(a: Int, b: Int): Int {
    return a + b
}
    ↓
// Kotlin IR   
IrSimpleFunction(name=add, returnType=Int)
  IrValueParameter(name=a, type=Int)
  IrValueParameter(name=b, type=Int)
  IrBlockBody
    IrReturn
      IrCall(symbol=Int.plus)
        IrGetValue(a)
        IrGetValue(b)
    ↓
// LLVM IR
define i32 @add(i32 %a, i32 %b) {
entry:
  %0 = add i32 %a, %b
  ret i32 %0
}
```

### 生成 BC 文件
BC 文件（Bitcode File）是 LLVM 中间表示（IR）的二进制序列化格式。它将 LLVM IR 以紧凑的二进制形式存储在磁盘上。

这一阶段的输入是经过 LTO (链接时优化)等优化的 LLVM IR，输出是二进制序列化的 BC 文件，供后续生成最终产物时使用。

```kotlin
/**
 * Write in-memory LLVM module to filesystem as a bitcode.
 */
internal val WriteBitcodeFilePhase = createSimpleNamedCompilerPhase<PhaseContext, WriteBitcodeFileInput>(
        "WriteBitcodeFile",
) { context, (llvmModule, outputFile) ->
    // Insert `_main` after pipeline, so we won't worry about optimizations corrupting entry point.
    insertAliasToEntryPoint(context, llvmModule)
    LLVMWriteBitcodeToFile(llvmModule, outputFile.canonicalPath)
}
```

### 编译和链接 (compileAndLink)
这一步就到生成最终产物的阶段了，在这个阶段主要做了以下工作：

- 通过 clang++ 将 BC 文件编译为目标文件(.o)
- 确定链接器输出类型，根据输入参数决定链接器产物是可执行文件还是静态库或者动态库
- 根据平台确定链接器，比如 mac 平台是 ld 链接器，链接所有依赖后最终生成可执行文件

```
BC 文件 → .o 文件 → 链接所有依赖 → 可执行文件
                        ↓
              静态缓存 + 动态缓存 + 系统库
```

```kotlin
internal val LinkerPhase = createSimpleNamedCompilerPhase<PhaseContext, LinkerPhaseInput>(
        name = "Linker",
) { context, input ->
    val linker = Linker(
            config = context.config,
            linkerOutput = input.outputKind,
            outputFiles = input.outputFiles,
            tempFiles = input.tempFiles,
    )
    val commands = linker.linkCommands(
            input.outputFile,
            input.objectFiles,
            input.dependenciesTrackingResult,
            input.resolvedCacheBinaries
    )
    runLinkerCommands(context, commands, cachingInvolved = !input.resolvedCacheBinaries.isEmpty())
}
```

## 总结
- 入口阶段
    - 从 K2Native.main() 开始，经过 doExecute() 和 KonanDriver.run()
    - 在 splitOntoTwoStages() 中决定采用两阶段编译
- 第一阶段：KLib 生成
    - FIR 生成：源代码 → 前端中间表示
    - IR 转换：FIR → 中间表示
    - IR 降级：预序列化处理
    - 序列化：生成包含 ir/, linkdata/, manifest 等的 KLib 文件
- 第二阶段：二进制生成
    - 前端分析：反序列化 KLib 获取模块描述符
    - IR 反序列化：从 KLib 重建 IR 模块
    - IR 降级：更抽象、高级的语言特性分解成更基础的元素
    - 代码生成：IR → LLVM IR 转换
    - BC 生成：LLVM IR 序列化为 Bitcode
    - 编译链接：BC → .o → 最终可执行文件


