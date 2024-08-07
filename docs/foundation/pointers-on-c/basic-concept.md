# 基本概念
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

毫无疑问，学习一门编程语言的基础知识不如编写程序有趣。但是，不知道语言的基础知识会使你在编写程序时缺少乐趣。

## 环境
在ANSI C的任何一种实现中，存在两种不同的环境，一种是编译环境，一种是执行环境

### 编译
编译阶段由几个步骤组成，组成一个程序的每个（有可能有多个）源文件通过编译过程分别转换为目标代码 (object code)。然后，各个目标文件由链接器 (linker)捆绑在一起，形成一个单一而完整的可执行程序。链接器同时也会引入标准C函数库中任何被该程序所用到的函数，而且它也可以搜索程序员个人的程序库，将其中需要使用的函数也链接到程序中。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/april/p13.png)

编译过程具体分为以下几个阶段：

- 预处理器 (preprocessor)处理
- 然后，源代码经过解析 (parse)，判断它的语句的意思
- 随后，便产生目标代码
- 如果在编译时加入优化相关命令，优化器 (optimizer)就会对目标代码进一步进行处理，使它效率更高

### 执行
程序的执行过程也需要经历几个阶段。

- 首先，程序必须载入到内存中，这个任务由操作系统完成。
- 然后，程序的执行便开始。通常一个小型的启动程序会与程序链接在一起。它负责处理一系列日常事务，如收集命名行参数以便使程序能够访问它们。接着，便调用main函数。
- 现在，便开始执行程序代码。在绝大多数机器里，程序将使用一个运行时堆栈 (stack)，它用于存储函数的局部变量和返回地址。程序同时也可以使用静态 (static)内存，存储于静态内存中的变量在程序的整个执行过程中将一直保留它们的值。
- 程序执行的最后一个阶段就是程序的终止，它可以由多种不同的原因引起。

## 词法规则
词法规则就像英语中的拼写规则，决定你在源程序中如何形成单独的字符片段，也就是标记 (token)。

一个 ANSI C 程序由声明和函数组成：

- 函数定义了需要执行的工作
- 而声明则描述了函数和（或）函数将要操作的数据类型（有时候是数据本身）。

### 注释
C语言的注释以字符/*开始，以字符*/结束，中间可以包含除*/之外的任何字符。在源代码中，一个注释可能跨越多行，但它不能嵌套于另一个注释中。注意，/*或*/如果出现在字符串字面值内部，就不再起注释定界符的作用。

所有的注释都会被预处理器拿掉，取而代之的是一个空格。因此，注释可以出现于任何空格可以出现的地方。

### 自由形式的源代码
C 是一种自由形式的语言，也就是说并没有规则规定什么地方可以书写语句，一行中可以出现多少条语句，什么地方应该留下空白以及应该出现多少空白等 。

唯一的规则就是相邻的标记之间必须出现一至多个空白字符(或注释)，不然它们可能被解释为单个标记。

### 标识符
标识符(identifier)就是变量、函数、类型等的名字。它们由大小写字母、数字和下划线组成，但不能以数字开头。

C是一种大小写敏感的语言，所以abc、Abc、abC和ABC是4个不同的标识符。标识符的长度没有限制，但标准允许编译器忽略第31个字符以后的字符。标准同时允许编译器对用于表示外部名字（也就是由链接器操纵的名字）的标识符进行限制，只识别前六位不区分大小写的字符。

### 程序的形式
一个C程序可能保存于一个或多个源文件中。虽然一个源文件可以包含超过一个的函数，但每个函数都必须完整地出现于同一个源文件中。

## 小结
一个C程序的源代码保存在一个或多个源文件中，但一个函数只能完整地出现在同一个源文件中。把相关的函数放在同一个文件内是一种好策略。每个源文件都分别编译，产生对应的目标文件。然后，目标文件被链接在一起，形成可执行程序。编译和最终运行程序的机器有可能相同，也可能不同。

程序必须载入到内存中才能执行。在宿主式环境中，这个任务由操作系统完成。在自由式环境中，程序常常永久存储于ROM中。经过初始化的静态变量在程序执行前能获得它们的值。你的程序执行的起点是main函数。绝大多数环境使用堆栈来存储局部变量和其他数据。

C编译器所使用的字符集必须包括某些特定的字符。如果你使用的字符集缺少某些字符，可以使用三字母词来代替。转义序列使某些无法打印的字符得以表达，例如在程序中包含某些空白字符。

注释以/*开始，以*/结束，它不允许嵌套。注释将被预处理器去除。标识符由字母、数字和下划线组成，但不能以数字开头。在标识符中，大写字母和小写字母是不一样的。关键字由系统保留，不能作为标识符使用。C是一种自由形式的语言。但是，用清楚的风格来编写程序有助于程序的阅读和维护

## 编程练习
### 练习 1
编写一个程序，它从标准输入读取C源代码，并验证所有的花括号都正确地成对出现。注意：你不必担心注释内部、字符串常量内部和字符常量形式的花括号。

```c
#include <stdio.h>

int main() {
    int c, braceCount = 0;

    while ((c = getchar()) != EOF) {
        if (c == '{') {
            ++braceCount;
        } else if (c == '}') {
            --braceCount;
            if (braceCount < 0) {
                // 更多的右花括号而不是左花括号
                printf("错误：多余的右花括号\n");
                return 1;
            }
        }
    }

    if (braceCount > 0) {
        // 更多的左花括号而不是右花括号
        printf("错误：未闭合的左花括号\n");
    } else {
        printf("所有花括号正确成对出现。\n");
    }

    return 0;
}
```




