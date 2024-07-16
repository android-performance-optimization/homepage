# 汇编编译器
前面介绍了机器语言的两种表现形式，即汇编形式和二进制形式。本章将介绍编译器如何系统地将汇编语言编写的程序翻译成二进制形式。

## 背景知识
符号化的语言称为汇编（assembly），翻译程序称为汇编编译器（assembler）。汇编编译器对每个汇编命令的所有部分进行解析，将每个部分翻译成它对应的二进制码，并将生成的二进制码汇编成真正能被硬件执行的二进制指令。

汇编编译器（Assembler） 汇编程序在被计算机执行之前，必须被翻译成计算机的二进制机器语言。翻译任务是由称为汇编编译器的程序来完成。汇编编译器的输入是一串汇编命令，然后产生一串等价的二进制指令作为输出。生成的代码被加载到计算机的内中然后被硬件执行。

可见，汇编编译器实际上主要是个文本处理程序，设计目的是用来提供翻译服务。写汇编编译器的程序员必须有完整的汇编语法说明文档和相应的二进制代码。有了这样约定（通常称为机器语言规范），就不难编写程序，让其对每个符号命令执行下面的任务：

- 解析出符号命令内在的域。
- 对于每个域，产生机器语言中相应的位域。
- 用内存单元的数字地址来替换所有的符号引用。
- 将二进制码汇编成完整的机器指令。

## 实现
Hack 汇编编译器的输入是名为 Prog.asm 的文本文件，该文件包含一个 Hack 汇编程序，编译器输出名为 Prog.hack的文本文件，该文件包含编译后的 Hack 机器代码。输入文件的名字作为一个命令行参数提供给汇编编译器。

我们这里提出一个基于 4 个模块的汇编编译器的实现：语法分析器（Parser）模块用来对输入文件进行语法分析：编码（Code）模块用来提供所有汇编命令所对应的二进制代码；符号表（SymbolTable）模块用来处理符号。另外还有一个主程序用来驱动整个编译过程。

- Parser：封装对输入代码的访问操作。功能包括：读取汇编语言命令并对其进行解析；提供“方便访问汇编命令成分（域和符号）”的方案；去掉所有的空格和注释。
- Code： 将 Hack 汇编语言助记符翻译成二进制码。
- SymbolTable：在符号标签（symbolic labels）和数字地址之间建立关联。
- Assembler: 驱动整个编译过程。

## 测试
将生成的汇编代码与预期的汇编代码进行比较，如果相同则测试通过。

```kotlin
class AssemblerTest {
    @Test
    fun testAdd() {
        val path = "testData/assembler/add/add.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/add/add.hack"
        val comparePath = "testData/assembler/add/add.hack.compare"
        compareFile(outputPath, comparePath)
    }

    @Test
    fun testMax() {
        val path = "testData/assembler/max/Max.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/max/Max.hack"
        val comparePath = "testData/assembler/max/Max.hack.compare"
        compareFile(outputPath, comparePath)
    }

    @Test
    fun testMaxL() {
        val path = "testData/assembler/max/MaxL.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/max/MaxL.hack"
        val comparePath = "testData/assembler/max/MaxL.hack.compare"
        compareFile(outputPath, comparePath)
    }

    @Test
    fun testRect() {
        val path = "testData/assembler/rect/Rect.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/rect/Rect.hack"
        val comparePath = "testData/assembler/rect/Rect.hack.compare"
        compareFile(outputPath, comparePath)
    }

    @Test
    fun testRectL() {
        val path = "testData/assembler/rect/RectL.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/rect/RectL.hack"
        val comparePath = "testData/assembler/rect/RectL.hack.compare"
        compareFile(outputPath, comparePath)
    }

    @Test
    fun testPong() {
        val path = "testData/assembler/pong/Pong.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/pong/Pong.hack"
        val comparePath = "testData/assembler/pong/Pong.hack.compare"
        compareFile(outputPath, comparePath)
    }

    @Test
    fun testPongL() {
        val path = "testData/assembler/pong/PongL.asm"
        Assembler().assemble(path)
        val outputPath = "testData/assembler/pong/PongL.hack"
        val comparePath = "testData/assembler/pong/PongL.hack.compare"
        compareFile(outputPath, comparePath)
    }

    private fun compareFile(outputPath: String, comparePath: String) {
        val output = File(outputPath).readText()
        val compare = File(comparePath).readText()
        assert(output == compare)
    }

}
```