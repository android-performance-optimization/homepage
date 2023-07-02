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

比如我们上面的输出，最开始的 4 个字节是所有 ELF 文件都必须相同的标识码：`0x7f`, `0x45`, `0x4c`, `0x46`, 第一字节对应 DEL 控制符的 ASCII 码，后面 3 个字节则正好是 ELF 三个字母的 ASCII 码

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
接下来我们通过 objdump 命令看看数据段与只读数据段的内容

```
$ objdump -x -s -d SimpleSection.o

...

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  1 .data         00000008  0000000000000000  0000000000000000  000000a0  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  3 .rodata       00000004  0000000000000000  0000000000000000  000000a8  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA

Contents of section .data:
 0000 54000000 55000000                    T...U...        
Contents of section .rodata:
 0000 25640a00                             %d..                
```

.data 段保存的是那些已经初始化了的全局静态变量和局部静态变量，`SimpleSection.c`中的`global_init_var`与`static_var`两个变量属于这种情况，这两个变量每个4个字节，一共刚好 8 个字节，所以 .data 段的大小为8个字节

.rodata 段存放的是只读数据，一般是程序中的只读变量(如 const 修饰的变量)和字符串常量，比如在`SimpleSection.c`中调用`printf`时用到的字符串常量，就储存在 .rodata 段中。需要注意的是，有的编译器会把字符串常量放在 .data 段中，而不会单独放在 .rodata 段中

接下来我们看下两个段中存储的内容

.data 中存储的内容即`0x00000054`与`0x00000055`，以小端序存放，它们的值正好对应十进制的`84`与`85`，也就是我们赋给`global_init_var`与`static_var`的值

.rodata 中存储的内容为`0x25640a00`，正好对应`%d\n`的 ASCII 码，可以对照[ASCII码一览表，ASCII码对照表](http://c.biancheng.net/c/ascii/)查看

### .bss 段
.bss 段存放未初始化的全局变量和局部静态变量，那么问题来了，为什么要把已初始化的变量和未初始化的变量分开存储的，为什么不直接放在 .data 段中？

答案是 .bss 段不占空间，我们接下来看一个直观的例子，来看看 .bss 的作用

```
$ echo "char array[1024*1024*64] = {'A'}; int main() {return 0;}" | gcc -x c - -o data
$ ls -lh data
-rwxrwxrwx 1 codespace codespace 65M Jun 25 01:26 data
$ echo "char array[1024*1024*64]; int main() {return 0;}" | gcc -x c - -o bss
$ ls -lh bss
-rwxrwxrwx 1 codespace codespace 17K Jun 25 01:27 bss
```

- 示例 1 中，array 变量已经被初始化，存放在 .data 段中，占用文件空间，因此整个文件大小共有 65 M
- 示例 2 中，array 变量未被初始化，存放在 .bss 段中，不占用文件空间，因此整个文件大小只有 17 K

可以看到，差别非常大。当然 .bss 段不占据实际的磁盘空间，但它的大小与符号还是要有地方存储，.bss 段的大小记录在段表中，符号记录在符号表中。当文件加载运行时，才分配空间以及初始化

接下来我们用 objdump 命令来看看`SimpleSection.o` 的 .bss 段的内容

```
$ objdump -x -s -d SimpleSection.o

...
Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  2 .bss          00000004  0000000000000000  0000000000000000  000000a8  2**2
                  ALLOC
SYMBOL TABLE:
0000000000000000 l     O .bss   0000000000000004 static_var2.1921
0000000000000004       O *COM*  0000000000000004 global_uninit_var
```

可以看到，我们本来预期 .bass 段中会有`global_uninit_var`与`static_var2`两个变量，共 8 个字节，实际上只有`static_var2`一个变量，4 个字节

这是因为有些编译器会将全局的未初始化变量存放在目标文件.bss 段，有些则不存放，只是预留一个未定义的全局变量符号，等到最终链接成可执行文件的时候再在 .bss 段分配空间

### 其他段
除了 .text, .data, .bss 这 3 个最常用的段之外，ELF 文件也包含一些其他的段，下面列出了一些常见的段

| 段名 |说明  |
| --- | --- |
|  .rodata1| 只读数据段，存放只读数据，与 .rodata 一样 |
|  .comment| 存放编译器版本信息 |
|  .debug| 调试信息 |
|  .dynamic| 动态链接信息 |
|  .hash| 符号哈希表 |
|  .line| 调试时的行号表，即源代码行号与编译后指令的对应表 |
|  .note| 额外的编译信息，如程序的公司名，发布版本号等 |
|  .strtab| 字符串表，用于存储 ELF 中的各种字符串 |
|  .symtab | 符号表 |
|  .shstrtab| 段表字符串表，用于存储段表中用到的字符串 |
|  .plt .got | 动态链接的跳转表和全局入口表 |
|  .init .finit | 程序初始化与终结代码段 |
|  .rel.text | 重定位表 |

这里面的很多段我们之后都会用到，比如 PLT Hook 中会用到的 .plt .got 段，在静态链接中会用到重定位表，这里可以先留个印象

## Elf 中的符号
链接过程的本质是把多个目标文件按照一定的规则拼接起来，在链接过程中，目标文件的拼接其实就是目标文件之间对地址的引用，即对函数和变量的地址的引用。

每个函数或变量都有自己独特的名字，才能避免链接过程中不同变量和函数之间的混淆。在链接中，我们将函数和变量统称为符号（Symbol），函数名或变量名就是符号名（Symbol Name）。

整个链接过程正是基于符号才能够正确完成。链接过程中很关键的一部分就是符号的管理，每一个目标文件都会有一个相应的符号表（Symbol Table），这个表里面记录了目标文件中所用到的所有符号。每个定义的符号有一个对应的值，叫做符号值（Symbol Value），对于变量和函数来说，符号值就是它们的地址

我们将符号表中的符号分为以下几类:

- 定义在本目标文件的全局符号，可以被其他目标文件引用。比如`SimpleSection.o`里面的`func1`、`main`和`global_init_var`。
- 在本目标文件中引用的全局符号，却没有定义在本目标文件，这一般叫做外部符号（External Symbol），也就是我们前面所讲的符号引用。比如`SimpleSection.o`里面的`printf`。
- 段名，这种符号往往由编译器产生，它的值就是该段的起始地址。比如`SimpleSection.o`里面的`.text`、`.data`等。
- 局部符号，这类符号只在编译单元内部可见。比如`SimpleSection.o`里面的`static_var`和`static_var2`。调试器可以使用这些符号来分析程序或崩溃时的核心转储文件。这些局部符号对于链接过程没有作用，链接器往往也忽略它们。
- 行号信息，即目标文件指令与源代码中代码行的对应关系，它也是可选的。”

其中最值得关注的就是全局符号，因为链接过程只关心全局符号的相互拼接，局部符号、段名、行号等都是次要的，它们对于其他目标文件来说是“不可见”的，在链接过程中也是无关紧要的

### ELF 符号表的结构
首先我们通过 readelf 命令来查看`SimpleSection.o` 的符号表

```
$ readelf -s SimpleSection.o

Symbol table '.symtab' contains 18 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS SimpleSection.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    3 static_var.1920
     7: 0000000000000000     4 OBJECT  LOCAL  DEFAULT    4 static_var2.1921
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT    9 
    11: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
    12: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 global_init_var
    13: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM global_uninit_var
    14: 0000000000000000    40 FUNC    GLOBAL DEFAULT    1 fun1
    15: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    16: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    17: 0000000000000028    55 FUNC    GLOBAL DEFAULT    1 main
```

接下来我们介绍一个各个列的含义，如下表所示

| 字段 |  含义|
| --- | --- |
| Name | 符号名 |
| Value | 符号对应的值，不同的符号，其值的含义不同，见下文详细解析 |
| Size | 符号大小，对于包含数据的符号，这个值是数据类型的大小，比如一个 int 类型的符号占 4 个字节，如果该值为 0 表示该符号大小为 0 或未知 |
| Type | 符号类型，分为未知符号类型，数据对象类型，函数类型，文件类型等 |
| Bind | 绑定信息，用于区分局部符号，全局符号与弱引用符号 |
| Vis | 在 C/C++ 中未使用，可忽略 |
| Ndx | 符号所在的段,如果符号定义在本目标文件中，该值表示符号所在段在段表中的下标。<br>如果该值为 ABS,表示该符号包含了一个绝对的值，比如上面的文件类型的符号。<br>如果该值为 COM，表示该值是一个 Common 块类型的符号。<br>如果该值为 UND，表示为定义，说明该符号在本目标文件中被引用，在其他文件中声明 |

上面说到不同的符号，其值含义不同，具体可以分为以下几种

- 在目标文件中，如果是符号的定义并且该符号不是`COMMON块`类型的，则`Value`表示该符号在段中的偏移。比如`SimpleSection.o`中的`func1`、`main`和`global_init_var`。
- 在目标文件中，如果符号是`COMMON块`类型的，则`Value`表示该符号的对齐属性。比如`SimpleSection.o`中的`global_uninit_var`。
- 在可执行文件中，`Value`表示符号的虚拟地址。这个虚拟地址对于动态链接器来说十分有用。

### C++ 的 Name Mangling 机制
我们前面提到每个函数或变量都有自己独特的名字，才能避免链接过程中不同变量和函数之间的混淆，因此在 C 语言目标文件链接过程中，如果有两个文件中都有`fun1`函数，链接过程就会报错

但当程序很大时，不同的模块由多人开发，如果命名规范不严格，很容易出现符号冲突的问题，于是像C++这样的后来设计的语言开始考虑到了这个问题，增加了名称空间（Namespace）的方法来解决多模块的符号冲突问题。

同时 C++拥有类、继承、虚机制、重载、名称空间等这些特性，它们使得符号管理更为复杂。最简单的例子，两个相同名字的函数`func(int)`和`func(double)`，尽管函数名相同，但是参数列表不同，那么编译器和链接器在链接过程中如何区分这两个函数呢？为了支持 C++ 这些复杂的特性，人们发明了符号修饰（Name Decoration）机制

比如下面这段代码

```c++
int    func(int i)           { return 0;     }
float  func(int i, float f)  { return i + f; }
double func(int i, double d) { return i+d;   }
```

经过name mangling操作后，得到的符号表中和`func`有关的内容如下：

```
$ g++ main.cc -o main.o && objdump -t main.o
main.o:     file format elf64-x86-64

SYMBOL TABLE:
0000000000001157 g     F .text  000000000000001c              _Z4funcid
000000000000113b g     F .text  000000000000001c              _Z4funcif
0000000000001129 g     F .text  0000000000000012              _Z4funci
...

```

可以看到，所有的符号都以`_Z`开头，前缀`_Z`是 GCC 的规定，具体是怎样转化的这里就不详细介绍了，有兴趣的读者可以参考GCC的名称修饰标准。同时我们也可以利用 nm 或 c++filt 等工具来解析被修饰的符号，不用自己手动解析

Name Mangling 机制使用地非常广泛，当我们查看 android so 的符号表时，可以看到很多以`_Z`开头的符号，就可以知道他们都是被修饰过的符号

## 总结
本文详细介绍了 ELF 文件的详细结构，包括文件头，段表，各个段的结构，符号表的结构等内容。

这些基础知识可能有些枯燥，但在 Android 性能优化方面的应用却相当广泛，因此还是有必要了解一下这些知识点的