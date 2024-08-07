# 机器语言
我们可用结构化（constructively）的方式来描述计算机，给出它的硬件平台，然后解释它是如何由一些底层芯片构建而成的。也可以通过描述和演示机器语言的功能来抽象地描述计算机。事实上，要开始了解一个新计算机系统，先去阅读一些用其机器语言编写的底层程序是十分必要的。这样不仅有助于了解如何通过程序让计算机去执行正确的操作，还可以由此了解为什么其硬件会按某种特定的方式来设计。

机器语言是一种约定的形式，用来对底层程序进行编码，从而形成一系列机器指令。应用这些指令，程序员可以命令处理器执行算术和逻辑操作，在内存中进行存取操作，让数据在寄存器之间传递，验证布尔表达式的值，等等。高级语言的基本设计目标是通用性和较强的表达力，机器语言的设计目标是能直接运行在指定的硬件平台上，并能对这个平台进行全面操控。当然，通用性、表达力和良好的结构仍然是必要的，但这些是仅相对于它所支持并直接运行于其上的硬件平台而言的。

## 背景知识
### 机器
#### 机器语言（machine language）
可以被看作是一种约定的形式，它利用处理器和寄存器来操控内存。

#### 内存 
内存（memory）的概念是指“用来储存数据和指令的硬件设备”。

从程序员的观点来看，所有的内存具有相同的结构：一个连续的固定宽度的单元序列，也称为字（word）或内存单元，每个内存单元都一个唯一的地址（address）。因此，对于独立的字（word，
代表一个数据项或是一个指令），可以通过提供它的地址来描述。

#### 处理器 
处理器，通常又称为中央处理器或 CPU（Central Processing Unit），是执行一组固定基本操作的设备。

这些操作通常包括算术操作和逻辑操作，内存存取操作和控制操作（control operation，也称 branching operations）。这些操作的对象是二进制数值，它们来自寄存器和指定的内存单元。类似的，操作的结果（处理器的输出）既可以存储在寄存器内，也可以存储在指定的内存单元。

#### 寄存器 
内存访问是相对较慢的操作，需要很长的指令格式（一个地址可能需要32位）。基于此原因，大多数处理器都配有一些寄存器，每个寄存器只存储1 位。它紧挨着处理器，相当于处理器的一个高速本地内存，使得处理器能快速地操控数据和指令。寄存器使得程序员能够尽可能地使用内存访问命令，从而加速程序的执行。

### 语言
机器语言程序是一系列的编码指令。比如说，在16-位计算机上的典型指令之一是 1010001100011001。为了知道这个指令的意思，就必须知道语言的规则，也就是底层硬件平台的指令集。例如，这样的指令包含四个4比特的位域（fields）：最左边的域是CPU的操作编码，剩下的三个部分表示该操作的操作数。因此根据该硬件平合上的机器语言语法，前面的命令代表 set R3 to R1 + R9。

鉴于二进制码相当晦涩，通常会在机器语言中同时使用二进制码和助记符（symbolicmnemonics）。例如，语言设计者可定义操作码 1010 用add来表示，机器中的寄存器可以使用符号RO、R1、R2等来表示。

将这种符号抽象更进一步发展，我们不仅能够阅读符号表示，而且能实际地利用这些符号命令而不是二进制指令来编写程序。接下来，可以使用文本处理程序，将这些符号命令解析为其内含的意域（助记符或操作数），将每个意域翻译成其对应的二进制表示，然后将生成的代码汇编成二进制机器指令。符号表示也称为汇编语言 （assembly language），或简单地说成汇编，而将汇编程序翻译成二进制码的程序则称为汇编编译器（assembler）。

### 命令
下面介绍所有机器语言都支持的通用命令集合

#### 算术操作和逻辑操作 
计算机需要执行基本的算术操作（比如加法和减法），以及基本的布尔操作（比如按位取反、移位等等）。

#### 内存访问 
内存访问命令分为两类。

第一类，正如刚才看到的，算术命令和逻辑命令不仅允许操控寄存器，而且还可以操控特定的内存单元。

第二类，所有的计算机都会使用 load 和 store 命令，用来在寄存器和内存之间传递数据。这些访问命令可能会应用某些类型的寻址方式，并在指令中指定目标内存单元的地址。当然，不同的计算机有不同的寻址方式。

下面列出三种绝大多数计算机都支持的寻址方式：

- 直接寻址（Direct Addressing） 最常用的寻址方式，直接表示一个指定内存单元的地址，或者使用一个符号来代表这个指定的地址。
- 立即寻址（Immediate Addressing） 这种寻址方式被用来加载常数，也就是说，加载那些出现在指令代码里面的数值：我们直接将指令数据域中的内容当作要操作的数
据装入寄存器，而不是将该数值当作内存单元的地址。
- 间接寻址（Indirect Addressing）在这种寻址模式中，要访问的内存单元的地址没有直接出现在指令中，而是指令指定的内存单元中的内容代表目标内容单元的地址。这种寻址模式被用来处理指针 （pointer）这种语言设施。

#### 控制流程
程序通常以线性方式，一个命令接着一个命令执行，但偶尔也包含分支，执行其他地方的命令。分支能够实现好几种结构，包括反复（repetition，跳回到循环的初始位置）、有条件的执行（conditional execution，如果布尔条件是false，则向前跳到“if-then”语句之后的位置），以及子程序调用（subroutine calling，跳到另一代码段的第一条命令处）。为了支持这些程序结构，各种机器语言都可以有条件（conditional）或无条件（unconditional）地跳转到程序指定的地址。

## Hack 机器语言规范
### 概述
Hack 是一个基于冯•诺伊曼架构的16-位计算机，由一个CPU、两个独立的内存模块（instruction memory 即指令内存和 data memory 即数据内存），以及两个内存映射1/O设备
（显示器和键盘）组成。

#### 内存地址空间
Hack 程序员要了解，有两个不同的地址空间：指令地址空间（instruction memory，以下称为指令内存），数据地址空间（data memory，以下称为数据内存）。

两个内存区都是16-位宽，有15-位地址空间，这意味着两个内存可用的最大地址都是32K 的 16-bit word。

#### 寄存器（Registers）
Hack 程序员要接触两个称为D和A的16-位寄存器。这些寄存器能够被算术和逻辑指令显式地操控，比如 A=D-1或 D=！A 。

D 仅用来储存数据值，A既可作为数据寄存器也可作为地址寄存器。也就是说，根据指令的上下文含义，A 中的内容可以被解释为数值或数据存储器中的地址，或者作为
指令存储器中的地址。

### A 指令
A-指令用来为 A 寄存器设置15-位的值。

A-指令主要有三种不同的用途。首先，在程序控制下，它提供了唯一一种“将常数输入计算机”的方法：其次，通过将目标数据内存单元的地址放入 A 寄存器，来为将对该内
存单元进行操作的C-指令提供必要的条件；其三，通过将跳转的目的地址放入A 寄存器来为执行跳转的C-指令提供条件。

### C 指令
C-指令是 Hack 平台程序的重点，几乎包含要做的所有事情。

指令代码的描述可以说是对以下三种问题的回答：（a）计算什么；（b）将计算后的值存储到什么地方；（c）下一步做什么。

随同 A-指令一起，这些描述几乎决定了计算机所有可能的操作。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/july/p1.png)

指令编码最左边起第一位为1，则表示该指令是C-指令。接着的两位没有被使用。剩下的位构成了三个域，分别代表指令符号表述的三个部分。

符号指令 dest = comp:jump 的終个语义可以如下表示。

- Comp 域告诉 ALU 计算什么。
- Dest 域指明计算后的结果（ALU的输出）将被存储到什么地方。
- Jump 域描述了转移条件，即接下来要取出并执行哪一条命令。

#### Computation 规范(计算规范)
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/july/p2.png)

#### Destination 规范(目的地规范)
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/july/p3.png)

#### Jump 规范(跳转规范)
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/july/p4.png)

### 符号
汇编命令可以使用常数或符号来表示内存单元位置（地址）。通过以下三种方式应用到汇编语言中：

- 预定义符号：包括虚拟寄存器，预定义指针，I/O 指针等。
- 标签符号
- 变量符号

## 项目
### 乘法程序
```
// Multiplies R0 and R1 and stores the result in R2.
// (R0, R1, R2 refer to RAM[0], RAM[1], and RAM[2], respectively.)

// Put your code here.
// SET R2=0
@R2
M=0

// RO R1 > 0 
@R0
D=M
@END
D;JLE
@R1
D=M
@END
D;JLE

// SET R3=R1
@R1
D=M
@R3
M=D

// LOOP BODY 
(LOOP)
@R0
D=M
@R2
M=M+D
@R3
M=M-1
@R3
D=M
// R3=0 JUMP END
@END
D;JEQ
// R3>0 JUMP LOOP
@LOOP
D;JGT

// END PROGRAM
(END)
@END
0;JMP
```

### I/O 处理程序
```
// Runs an infinite loop that listens to the keyboard input. 
// When a key is pressed (any key), the program blackens the screen,
// i.e. writes "black" in every pixel. When no key is pressed, the
// program clears the screen, i.e. writes "white" in every pixel.

// Put your code here.
@i
M=0

(LOOP)
@KBD
D=M
@BLACK
D;JGT    // key>0 JUMP BLACK
@WHITE
D;JEQ    // key=0 JUMP WHITE

(BLACK)
@i
D=M
@8191
D=D-A
@LOOP
D;JGT   // if MAX Then do noting
@i
D=M
@SCREEN
A=D+A
M=-1
@i
M=M+1
@LOOP
0;JMP

(WHITE)
@i
D=M
@RESET
D;JLT   // if MIN Then do noting
@i
D=M
@SCREEN
A=D+A
M=0
@i
M=M-1
@LOOP
0;JMP

(RESET)
@i
M=0
@LOOP
0;JMP
```