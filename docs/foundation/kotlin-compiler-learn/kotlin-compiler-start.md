# Kotlin 编译器源码阅读
### 产物构建
通过执行 `./gradlew dist` 命令，可以将编译器分发到 `dist/kotlinc/` 目录下。

### 如何调试 Kotlin 编译器
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

### Kotlin/JS 编译器调试
```
export KOTLIN_COMPILER=org.jetbrains.kotlin.cli.js.K2JSCompiler

DIR="${BASH_SOURCE[0]%/*}"
: ${DIR:="."}

"${DIR}"/kotlinc "$@"
```

通过以上代码可知，`org.jetbrains.kotlin.cli.js.K2JSCompiler` 是 Kotlin/JS 编译器的主类，我们可以通过修改`K2JSCompiler`类，运行它的`main`方法，即可启动与调试 Kotlin/JS 编译器。

需要注意的是，K2 版本编译器不还不支持直接把 Kotlin 代码编译成 JS 代码，因此需要先把代码编译成 Klib，再把 Klib 编译成 JS 代码，同时 outputDir 不支持相对路径，只支持绝对路径，如下所示

```
#!/bin/sh

set -e

KOTLIN_HOME= # it should point to Kotlin home directory
BUILD_DIR=`pwd`/build # it must be absolute path

rm -rf $BUILD_DIR
mkdir -p $BUILD_DIR/libs # there should be two separate directories for KLIBs and JS
mkdir -p $BUILD_DIR/app

## first, build KLIB
$KOTLIN_HOME/bin/kotlinc-js \
  -Xir-produce-klib-file \
  -libraries $KOTLIN_HOME/lib/kotlin-stdlib-js.klib \
  -ir-output-dir $BUILD_DIR/libs \
  -ir-output-name main \
  main.kt

## then, build JS
$KOTLIN_HOME/bin/kotlinc-js \
  -Xir-produce-js \
  -libraries $KOTLIN_HOME/lib/kotlin-stdlib-js.klib \
  -Xinclude=$BUILD_DIR/libs/main.klib \
  -ir-output-dir $BUILD_DIR/app \
  -ir-output-name app
```

因此我们可以通过修改 preloader 类，运行它的 main 方法，即可启动与调试 Kotlin/JS 编译器。

```
String[] testArgs = {"-cp",  "./dist/kotlinc/lib/kotlin-compiler.jar", "org.jetbrains.kotlin.cli.js.K2JSCompiler",
                    "-libraries","./dist/kotlinc/lib/kotlin-stdlib-js.klib",
                    "-ir-output-dir","/Users/xxx/AndroidProject/leo/kotlin/compilerTestData","-ir-output-name","TestKotlin",
                    "-Xir-produce-klib-file",
                    "./compilerTestData/Test.kt",
            };
            run(testArgs);
```