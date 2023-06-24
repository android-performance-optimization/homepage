# Elf 文件格式详解
最近在读《程序员的自我修养：链接，装载与库》，其实这本书跟 Android 开发的联系还挺紧密的，无论是 NDK 开发，或者是性能优化中一些常用的 Native Hoook 手段，都需要了解一些链接，装载相关的知识点。本文为读书笔记。

## ELF 文件是什么？
ELF 即 Executable and Linkable Format，是 linux 下的可执行文件。

除了 ELF 文件本身，源代码编译后但未链接的中间文件(.o 文件)，动态链接库(.so 文件)，静态链接库(.a 文件)，都按照 ELF 文件格式存储.

ELF 文件标准中把系统中采用 ELF 格式的文件分为以下 4 类

- 可重定位文件(relocatable file)：包括 .o 文件和 .a 文件
- 可执行文件(executable file)：即 EFL 可执行文件，通常没有后缀
- 共享库文件(shared object file)：即 .so 文件
- 核心转储文件(core dump file): 即 core dump 文件

## ELF 文件总体结构
![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/june/p2.png)

一个 ELF 文件的总体结构如上图所示（上图为链接视图，执行视图略有不同），主要包括以下内容

- ELF Header，ELF文件头，它包含了描述整个文件的基本属性
- ELF 文件中的各个段(section)
- 段表(section header table), 该表描述了 ELF 文件包含的所有段的信息，比如每个段的段名，段的长度等
- 其他一些辅助结构，如字符串表，符号表等

## ELF 文件结构详解
在上面我们了解了 ELF 文件的总体结构，但耳听为虚，眼见为实，我们接下来实操一下，看下 Elf 文件具体是怎么样的

首先我们添加一个`SimpleSection.c`文件，如下所示：

```c
int printf(const char *format, ...);

int global_init_var = 84;
int global_uninit_var;

void fun1(int i)
{
    printf("%d\n", i);
}

int main(void)
{
    static int static_var = 85;
    static int static_var2;

    int a = 1;
    int b;
    fun1(static_var + static_var2 + a + b);
    return a;
}
```

接下来我们通过`gcc -c SimpleSection.c`命令只编译不链接生成目标文件：SimpleSection.o，这也是一个 ELF 文件，我们接下来就来分析这个文件的内容

### 文件头
```
$ readelf -h SimpleSection.o

ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          1184 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         14
  Section header string table index: 13
```

如上所示，通过`readelf`命令获取了文件头，可以看到，ELF 的文件头中定义了ELF魔数、文件机器字节长度、数据存储方式、版本、运行平台、ABI版本、文件类型、硬件平台、硬件平台版本、入口地址、程序头入口和长度、段表的位置和长度及段的数量等信息，包含了描述整个文件的基本属性

ELF 文件头结构及相关常数被定义在`/usr/include/elf.h`文件中，readelf 输出的信息与这些结构很多都可以一一对应，感兴趣的可以自行查看

#### ELF 魔数
头文件中比较有意思的是魔数，魔数的作用是用来确认文件的类型，操作系统在加载可执行文件时会检验魔数是否正确，如果不正确则会拒绝加载

比如我们上面的输出，最开始的 4 个字节是所有 ELF 文件都必须相同的标识码：0x7f,0x45,0x4c,0x46, 第一字节对应 DEL 控制符的 ASCII 码，后面 3 个字节则正好是 ELF 三个字母的 ASCII 码

第 5 个字节用于标识 ELF 文件类，0x01 表示 32 位，0x02 表示 64 位，第 6 个字节用于标记字节序，规定该 ELF 文件是大端还是小端的，第 7 个字节用于标记 ELF 文件主版本号，一般是 1

而后面的 9 个字节，ELF 标准还没有定义，一般填0，有些平台会使用这 9 个字节作为扩展标志

### 段表
在头文件之后就是 ELF 文件中各种各样的段了，我们通过 readelf 命令来查看段表，段表中记录了每个段的段名，段的长度，在文件中的偏移，读写权限，以及其它属性

```
$ readelf -S SimpleSection.o

There are 14 section headers, starting at offset 0x4a0:

Section Headers:
  [Nr] Name              Type             Address           Offset    Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000  0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040  000000000000005f  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000380  0000000000000078  0000000000000018   I      11     1     8
  [ 3] .data             PROGBITS         0000000000000000  000000a0  0000000000000008  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  000000a8  0000000000000004  0000000000000000  WA       0     0     4
  [ 5] .rodata           PROGBITS         0000000000000000  000000a8  0000000000000004  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  000000ac  000000000000002c  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000d8  0000000000000000  0000000000000000           0     0     1
  [ 8] .note.gnu.propert NOTE             0000000000000000  000000d8  0000000000000020  0000000000000000   A       0     0     8
  [ 9] .eh_frame         PROGBITS         0000000000000000  000000f8  0000000000000058  0000000000000000   A       0     0     8
  [10] .rela.eh_frame    RELA             0000000000000000  000003f8  0000000000000030  0000000000000018   I      11     9     8
  [11] .symtab           SYMTAB           0000000000000000  00000150  00000000000001b0  0000000000000018          12    12     8
  [12] .strtab           STRTAB           0000000000000000  00000300  000000000000007b  0000000000000000           0     0     1
  [13] .shstrtab         STRTAB           0000000000000000  00000428  0000000000000074  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

可以看出，段表是一个数组，数组的每一项对应每一个段，其中第一个项是无效的，类型为`NULL`，因此`SimpleSection.o`共有 13 个有效的段，段表对应的结构体也定义在`/usr/include/elf.h`文件中，感兴趣的可以自行查看

下面我们介绍一下段表各个字段的含义

| 字段 |含义  |
| --- | --- |
| Name | 段名，段名是个字符串，位于一个叫作 .shstrtab 的字符串表中 |
| Type | 段的类型，段名只在编译和链接阶段有作用，不能真正地表示段的类型，我们也可以将一个数据段命名为".txt"，对于编译器和链接器来说，主要决定段的属性的是段的类型与段的标志位|
| Flags | 段的标志位，段的标志位表示该段在进程虚拟地址空间中的属性，比如是否可写，是否可执行等 |
| Address | 段虚拟地址，如果该段可以被加载，则为该段被加载后在进程地址空间中的虚拟地址，否则为 0 |
| Offset | 段偏移，如果该段存在于文件中，则表示该段在文件中的偏移，否则无意义，比如对于 .bss 段就没有意义 |
| Size | 段的长度 |
| Link 和 Info | 段的链接信息，如果段的类型是与链接相关的，比如重定位表，符号表等，则该字段有意义，否则无意义 |
| Align | 段对齐地址，有些段对段地址对齐有要求，由于地址对齐的数量都是 2 的指数倍，Align 表示对齐数量中的指数，比如当 Align = 3 时表示 8 倍，当 Algin 为 0 或者 1 时表示没有对齐要求 |
| EntSize | 项的长度，有些段包含了一些固定大小的项，比如符号表，它包含的每个符号所占的大小是一样的，对于这种段，EntSize 表示每一项的大小。如果为 0 表示该段没有固定大小的项 |

在了解了段表的结构之后，接下来我们看一下各个段的具体内容

### .text 代码段
首先我们通过 objdump 命令来看下代码段的具体内容，objdump 的 "-s" 参数可以将所有段的内容以 16 进制的方式打印出来，"-d" 参数可以将所有包含指令的段反汇编，结果如下所示：

```
$ objdump -s -d SimpleSection.o

Contents of section .text:
 0000 f30f1efa 554889e5 4883ec10 897dfc8b  ....UH..H....}..
 0010 45fc89c6 488d3d00 000000b8 00000000  E...H.=.........
 0020 e8000000 0090c9c3 f30f1efa 554889e5  ............UH..
 0030 4883ec10 c745f801 0000008b 15000000  H....E..........
 0040 008b0500 00000001 c28b45f8 01c28b45  ..........E....E
 0050 fc01d089 c7e80000 00008b45 f8c9c3    ...........E... 

Disassembly of section .text:

0000000000000000 <fun1>:
   0:   f3 0f 1e fa             endbr64 
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   48 83 ec 10             sub    $0x10,%rsp
   c:   89 7d fc                mov    %edi,-0x4(%rbp)
   f:   8b 45 fc                mov    -0x4(%rbp),%eax
  12:   89 c6                   mov    %eax,%esi
  14:   48 8d 3d 00 00 00 00    lea    0x0(%rip),%rdi        # 1b <fun1+0x1b>
  1b:   b8 00 00 00 00          mov    $0x0,%eax
  20:   e8 00 00 00 00          callq  25 <fun1+0x25>
  25:   90                      nop
  26:   c9                      leaveq 
  27:   c3                      retq   

0000000000000028 <main>:
  28:   f3 0f 1e fa             endbr64 
  2c:   55                      push   %rbp
  2d:   48 89 e5                mov    %rsp,%rbp
  30:   48 83 ec 10             sub    $0x10,%rsp
  34:   c7 45 f8 01 00 00 00    movl   $0x1,-0x8(%rbp)
  3b:   8b 15 00 00 00 00       mov    0x0(%rip),%edx        # 41 <main+0x19>
  41:   8b 05 00 00 00 00       mov    0x0(%rip),%eax        # 47 <main+0x1f>
  47:   01 c2                   add    %eax,%edx
  49:   8b 45 f8                mov    -0x8(%rbp),%eax
  4c:   01 c2                   add    %eax,%edx
  4e:   8b 45 fc                mov    -0x4(%rbp),%eax
  51:   01 d0                   add    %edx,%eax
  53:   89 c7                   mov    %eax,%edi
  55:   e8 00 00 00 00          callq  5a <main+0x32>
  5a:   8b 45 f8                mov    -0x8(%rbp),%eax
  5d:   c9                      leaveq 
  5e:   c3                      retq   
```

`Contents of section .text`就是.text的数据以十六进制方式打印出来的内容，总共 0x5f 字节，最左面一列是偏移量，中间4列是十六进制内容，最右面一列是 .text 段的 ASCII 码形式。

`Disassembly of section .text`则是代码段反汇编的结果，可以很明显地看到，.text 段中的内容就是`SimpleSection.c`里两个函数`func1()`和`main()`的指令。

### 数据段与只读数据段
```
$ objdump -x -s -d SimpleSection.o
```

### bss 段
存放未初始化的全局变量和局部静态变量

bss 不占空间：https://www.jianshu.com/p/52c7445af23a

https://www.zhihu.com/question/293002441

### 其他段


## Elf 中的符号
Elf 中的符号
