# Kotlin/Native 编译流程浅析
之前我们介绍了 Kotlin/JVM 的编译流程，今天我们来看看 Kotlin/Native 的编译流程。

## Kotlin/Native 编译器如何调试
- 构建 Kotlin/Native 编译器: ./gradlew :kotlin-native:dist
- 运行编译器: ./kotlin-native/dist/bin/run_konan  konanc ./compilerTestData/Hello.kt -o ./compilerTestData/Hello  -J"-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:50015"
- 上面命令中的-agentlib:jdwp=... 是 JVM 的调试参数，用于开启远程调试支持，运行以上命令后，可以使用 IDEA 附加调试到 50015 端口进行调试。

