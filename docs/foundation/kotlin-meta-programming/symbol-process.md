# 编译时的符号处理
本文为《深入实践 kotlin 元编程》读书笔记，想要深入了解 kotlin 元编程的同学可以阅读原书。

除了直接生成源代码，我们有时还会面临一种更加复杂的场景，即如何基于人工编写的源代码来生成代码，甚至还可以让二者相互调用。

为了应对这种需求场景，Java 和 Kotlin 编译器均提供了相应的扩展支持，允许我们编写一些特定的程序来访问源代码当中特定的语法结构，并完成代码的生成。这就是编译时的符号处理。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/april/p2.jpg)

## 符号的基本概念
### Java 的符号
Java 的符号包括变量(VarSymbol)、方法(MethodSymbol)、 类(ClassSymbol)、包(PackageSymbol)等等

Java 编译器提供了一套完善的符号处理的机制，即APT(Annotation Processing Tool，注解处理器)。不过，APT 的公开 API 当中并没有提到 Symbol，这是怎么回事呢?

这是因为 Java 编译器的设计者不希望暴露过多编译器内部的细节，基于符号的实现抽象出了以 Element 类型为基础的公开 API。

需要注意的是，Java 符号并不等同于 Java 抽象语法树(Abstract Syntax Tree， 缩写为 AST)节点，Java 语法树的节点类都是 JCTree 的子类，其中 JC 代表 Javac。与 Symbol 和 Type 相同，JCTree 也是 Java 编译器的内部实现，我们只需要清楚 Java 符 号是基于 Java 语法树创建出来的，它只包含语法树结构当中的类、方法、变量等结构，但不包含函数体、表达式内容等信息即可。

### Kotlin 的符号
Kotlin 的符号在 Google 的开源项目 KSP(Kotlin Symbol Processing，Kotlin 符号处理)当中给出，该项目目前已经成为 Kotlin 官方推荐的符号处理方案。

Kotlin 目前存在两套语法树实现，即 FE1.0 的 PSI 和 K2 编译器当 中的 FIR。与 Java 不同，Kotlin 的语法树的 API 并不算是严格意义的 内部 API，因为这些 API 已经被大量应用在编译器插件和 IntelliJ 插件的实现当中了。不过，由于 Kotlin 编译器还在不断地重写过程中，因此抽象出 Kotlin 符号的 概念来屏蔽 Kotlin 编译器的内部实现显然是一个更好的选择，这与 APT 的实现如出一辙。

Kotlin 符号的实现依赖于语法树的实现，二者从命名和用法上也有着非常接近的地方。Kotlin 的符号类型都实现自 KSNode 接口，例如类符号 KSClassDeclaration，函数符号KSFunction，等等，这些对应于 APT 当中的 Element 接口及其实现类。

与 Java 符号相同，Kotlin 的符号也是基于语法树节点构造出来的，只包含类、函数、 变量这样的符号信息，而不包含函数体、语句、表达式内容等等。

### 符号与语法树节点的关系和区别
前面已经提到，符号是基于语法树构造出来的，它只包含语法树当中的部分信息。确切地说，符号只包含程序的 ABI(Application Binary Interface，二进制接口)信息。

如果我们希望检查函数的命名是否符合代码规范，可以基于对符号的处理设计实现方案。但如果我们希望检查的是局部变量，那么只处理符号可能就有些力不从心了，因为符号当中只包含 类的成员变量而不包含局部变量。

## 处理器的基本结构
### APT 的基本结构
- 初始化
- 声明和配置处理器参数
- 声明处理器支持的注解类型
- 声明处理器支持的 Java 源码版本
- 核心处理方法

### KSP 的基本结构
KSP 的处理器只有三个函数:

- finish: 在处理器执行完成时调用。如果有需要关闭的资源，可以在这里处理。
- onError: 处理器执行过程中遇到错误，在执行完毕之后调用。
- process: 提供处理器的核心实现，通过读取符号信息，处理代码分析或者文件生成等逻辑。

### APT 与 KSP 的处理器的结构差异
KSP 没有像 APT 那样要求处理器明确给出支持的源码版本、注解类型和参数。KSP 的版本是与 Kotlin 编译器的版本绑定的，其中隐含了支持的最新的 Kotlin 源码版本。

APT 需要在处理器实现当中声明支持的注解类型，Java 编译器会在调用处理器之前根据源 码的实际情况来决定是否调用该处理器。相比之下，KSP 似乎并没有打算把自己限制于对注解的 处理范围之内。尽管它提供了很多与 APT 相似的 API，但在 KSP 当中直接获取所有的符号进行分析处理似乎是顺理成章的。

APT 要求在处理器当中声明需要的参数，这样使用者就能更清楚地知道处理器需要哪些参数。 然而，APT 没有对参数的传递做强制要求，因此不能对使用者形成有效的引导和约束。KSP 干 脆不要求处理器对参数做声明，处理器的开发者只需要在文档当中对参数做好说明并在处理器当中处理好参数缺省的情况即可。

### 处理器的配置文件
在了解了处理器的基本结构之后，我们还需要通过配置文件将处理器接入到编译流程当中， 让编译器在编译时可以加载并执行它们。由于 Java 和 Kotlin 编译器都是 JVM 程序，因此配置文件需要放到编译器运行时的 classpath 当中。

## 深入符号和类型
本节介绍如何获取符号当中的信息，这也是符号处理过程当中最核心的内容。

### 获取修饰符
获取符号的修饰符，在 APT 当中可以使用 Element#getModifiers()，这个方法适用于所有符号，因此如果我们希望判断一个类是不是公有的，只需要判断它的修 饰符集合当中是否包含 PUBLIC。

在 KSP 当中同样有类似的 API，即 KSModifierListOwner#modifiers，用法与 APT 类似。

### 通过名称获取相应的符号
在 APT 当中，我们可以使用 Elements#getTypeElement(CharSequence) 获取参数对应的符号，其中 Elements 的实例可以通过 ProcessingEnvironment 获取。

在 KSP 当中，可以通过 Resolver#getClassDeclarationByName(String) 来获取类符号，参数也可以是 KSName 类型。

我们还可以使用 Resolver#getFunctionDeclarationsByName(String, Boolean) 获取函数符号，使用Resolver#getPropertyDeclarationByName(String, Boolean) 来获取属性符号。

### 获取符号的类型
在 APT 当中，任何符号都可以通过 Element#asType() 方法来获取它的类型，返回值类型为 TypeMirror。

在 KSP 当中获取符号的类型也很容易。主要有两种情况，一种是通过类符号直接获取类型，另一种是通过类型引用符号解析类型。

那么，KSP 为什么要这么设计呢?

因为类型引用符号与其他符号一样，是源代码的直接表示，无需通过语义分析就可以获取到; 而解析类型引用符号是在所有类型中查找所引用的类型的过程，这个过程需要依赖语义分析的结果，有一定的开销。KSP 当中引入类型引用符号，使得类型的解析延迟到使用时，也是一种性能 上的优化。

### 通过类型获取符号
一个类型只有在代码当中定义之后才会出现在类型系统当中，因而任何类型都可以找到它对应的符号。

在 APT 当中，想要通过 TypeMirror 获取它对应的 Element，需要借助 Types#asElement() 方法，其中 Types 的实例可以在处理器的 init 方法当中通过 ProcessingEnvironment 获取。

在 KSP 当中，获取类型对应的符号可以直接使用 KSType#declaration。

### 判断类型之间的关系
类型之间存在相同(Same)、可赋值(Assignable)、子类型(Subtype)等关系。

#### 相同类型
相同类型很容易理解，在 APT 当中可以使用 Types#isSameType(TypeMirror,TypeMirror) 来判断两个 TypeMirror 实例是否是相同的类型。

在 KSP 当中，判断两个类型是否相同非常直接，使用 equals 运算符即可。

#### 可赋值类型和子类型
可赋值类型和子类型的表述在 Kotlin 当中没有实质的区别，即在 Kotlin 当中如果类 型 A 的实例可以赋值给类型 B 的变量，那么 A 类型就是 B 类型的子类型。

不过在 Java 当中，可赋值类型和子类型却不完全相同。尽管我们也经常通过两个类型是 否可赋值来判断二者之间是否存在子类型关系，不过在 Java 当中这并不是绝对的。

因此，我们可以看到 APT 当中，Types 提供了 isAssignable 来判断类型的可赋值关 系，以及 isSubtype 来判断类型的父子关系。而在 KSP 当中就比较简单了，KSType 只有一个 isAssignableFrom 来判断类型的可赋值关系。

### 获取注解及其参数值
我们可以通过符号获取到标注于它之上的注解，并进而获取到其中的值。

如果使用 APT 来处理这段代码，我们可以通过 RoundEnvironment 和注解符号直接获取到被注解标注的符号，通过这些符号再获取注解的参数。如果注解处理器的运行时可见，我们可以直接使用 getElementsAnnotatedWith 来获取注解的参数。

KSP 当中同样也存在这两种相似的做法。如果注解在处理器运行时不可见，我们同样需要通过处理符号来获取注解的信息。

## 深入理解符号处理器

 
