# 快速上手
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

上一节我们了解了如何配置 C 语言的开发环境，这一节我们通过一个简单的例子来快速了解 C 语言。

## 通过一段代码快速了解 C 语言
这个程序从标准输入读取输入行，然后将每一行输入以及行中的某些部分输出到标准输出。第一个输入是一组列号，以负数结束。列号是成对出现的，指定了要打印的输入行的列范围。例如，0 3 10 12 -1 表示只打印列 0 到 3 和列 10 到 12。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define MAX_COLS 20    /* max # of columns to process */
#define MAX_INPUT 1000 /* max len of input & output lines */

int read_column_numbers(int columns[], int max);
void rearrange(char *output, char const *input, int n_columns, int const columns[]);

int main(void)
{
    int n_columns;          /* # of columns to process */
    int columns[MAX_COLS];  /* the columns to process */
    char input[MAX_INPUT];  /* array for input line */
    char output[MAX_INPUT]; /* array for output line */

    /*
    ** 读取输入的列编号
    */
    n_columns = read_column_numbers(columns, MAX_COLS);

    /*
    ** 读取、处理并打印剩余的输入行
    ** gets函数是一个C语言标准库函数，用于从标准输入（通常是键盘）读取一行文本，直到遇到换行符（'\n'）或文件结束符。
    ** 它将读取的字符（不包括换行符）存储到其参数所指向的字符数组中，并在最后添加一个字符串结束符'\0'。
    ** gets函数在读到文件结束符（EOF）时会返回NULL，因此这个循环会一直运行，直到用户输入EOF标志（在Windows中通常是Ctrl+Z，而在Unix/Linux系统中是Ctrl+D）。
    */
    while (gets(input) != NULL)
    {
        printf("Original input : %s\n", input);
        rearrange(output, input, n_columns, columns);
        printf("Rearranged line: %s\n", output);
    }

    return EXIT_SUCCESS;
}

/*
** 读取列编号列表，忽略超出最大值的列编号
*/
int read_column_numbers(int columns[], int max)
{
    int num = 0;
    int ch;

    /*
    ** 获取列编号，遇到EOF或者小于0的数字时停止，循环结束的条件具体包括以下三个
    ** 1. 输入的列编号数量达到最大值
    ** 2. scanf(%d", &columns[num])的返回值不为1，即不再读取到整数
    ** 3. 读取到的整数小于0
    */
    while (num < max && scanf("%d", &columns[num]) == 1 && columns[num] >= 0)
        num += 1;

    /*
    ** 确保列编号是成对出现的
    */
    if (num % 2 != 0)
    {
        puts("Last column number is not paired.");
        exit(EXIT_FAILURE);
    }

    /*
    ** 丢弃最后一个数字后面的字符
    */
    while ((ch = getchar()) != EOF && ch != '\n')
        ;

    return num;
}

/*
** 通过连接指定列的字符来处理一行输入。然后输出行以NUL结尾。
** 当数组名作为实参时，传给函数的实际上是一个指向数组起始位置的指针，也就是数组在内存中的地址
** 因此当不希望原数据不被改别时，需要添加 const 修饰
*/
void rearrange(char *output, char const *input, int n_columns, int const columns[])
{
    int col;        /* subscript for columns array */
    int output_col; /* output column counter */
    int len;        /* length of input line */

    len = strlen(input);
    output_col = 0;

    /*
    ** 处理每一对列编号
    */
    for (col = 0; col < n_columns; col += 2)
    {
        int nchars = columns[col + 1] - columns[col] + 1;

        /*
        ** 如果输入行不够长或者输出数组已满，我们就结束了。
        */
        if (columns[col] >= len || output_col == MAX_INPUT - 1)
            break;

        /*
        ** 如果输出数组中没有足够的空间，只复制能放下的部分。
        */
        if (output_col + nchars > MAX_INPUT - 1)
            nchars = MAX_INPUT - output_col - 1;

        /*
        ** 复制相关数据
        */
        strncpy(output + output_col, input + columns[col], nchars);
        output_col += nchars;
    }

    /*
    ** 添加NUL字符
    ** 在output字符串的output_col位置处插入一个字符串终止符（null terminator），用于标记字符串的结束。在C语言中，字符串是以'\0'（ASCII码为0的字符）结束的字符数组。这个终止符是必须的，因为它告诉字符** 串处理函数（比如printf、strlen等）字符串在哪里结束。
    */
    output[output_col] = '\0';
}
```

通过上面的例子，我们可以快速了解 C 语言的输入，输出，函数，传参，循环等知识点。

所有的 C 程序必须有一个 main 函数，它是程序执行的起点。函数的标量参数通过传值的方式进行传递，而数组名参数则具有传址调用的语义。字符串是一串由 NULL 字节结尾的字符，并且有一组库函数以不同的方式专门用于操纵字符串。printf 函数执行格式化输出，scanf 函数用于格式化输入，getchar 和putchar 分别执行非格式化字符的输入和输出。if 和 while 语句在 C 语言中的用途跟它们在其他语言中的用途差不太多。

## 编程练习
### 练习 1
“Hello world!”程序常常是C编程新手所编写的第1个程序。它在标准输出中打印Hello world!，并在后面添加一个换行符。

```c
#include <stdio.h>

int main()
{
    printf("Hello World\n");
    return 0;
}
```

### 练习 2
编写一个程序，从标准输入读取几行输入。每行输入都要打印到标准输出上，前面要加上行号。在编写这个程序时要试图让程序能够处理的输入行的长度没有限制。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUFFER_SIZE 1024 // 定义缓冲区大小

int main() {
    char buffer[BUFFER_SIZE];
    int line_number = 1;

    printf("请输入文本（CTRL+D 或 CTRL+Z 以结束）:\n");

    // 读取输入直到EOF
    while (fgets(buffer, BUFFER_SIZE, stdin)) {
        // 打印行号和输入行
        printf("%d: %s", line_number++, buffer);

        // 如果最后一个字符不是换行符，说明可能还有更多内容未读取完
        if (buffer[strlen(buffer) - 1] != '\n') {
            int ch;
            // 消耗剩余的字符直到遇到换行符，以完成这一行的读取
            while ((ch = getchar()) != '\n' && ch != EOF);
            if (ch == '\n') {
                printf("\n...这一行超过了缓冲区长度，仅显示部分内容\n");
            }
        }
    }

    return 0;
}
```

### 练习 3
编写一个程序，从标准输入读取一些字符，并把它们写到标准输出上。它同时应该计算checksum值，并写在字符的后面。

checksum(检验和)用一个singed char类型的变量进行计算，它初始为-1。当每个字符从标准输入读取时，它的值就被加到checksum中。如果checksum变量产出了溢出，那么这些溢出就会被忽略。当所有的字符均被写入后，程序以十进制整数的形式打印出checksum的值，它有可能是负值。注意在checksum后面要添加一个换行符。

```c
#include <stdio.h>

int main() {
    signed char checksum = -1; // checksum的初始值
    int ch; // 用于存储输入的字符

    printf("请输入字符，回车结束输入:\n");

    // 使用getchar()从标准输入读取字符直到EOF
    while ((ch = getchar()) != EOF && ch != '\n') {
        checksum += ch; // 将字符的值加到checksum中
    }
    checksum += '\n';
    // 打印checksum的值
    printf("\nChecksum: %d\n", checksum);

    return 0;
}
```

### 练习 4
编写一个程序，一行行地读取输入行，直至到达文件尾。算出每行输入行的长度，然后把最长的那行打印出来。为了简单起见，你可以假定所有的输入行均不超过1000个字符。

```c
#include <stdio.h>
#include <string.h>

#define MAX_LINE_LENGTH 1000  // 假定的最大输入行长度

int main() {
    int len;  // 当前行长度
    int max;  // 目前为止发现的最长行的长度
    char line[MAX_LINE_LENGTH];    // 当前的输入行
    char longest[MAX_LINE_LENGTH]; // 用于保存最长的行

    max = 0;
    while (fgets(line, MAX_LINE_LENGTH, stdin) != NULL) {
        len = strlen(line);
        // 如果当前行比之前记录的最长行要长，则更新最长行和最长长度
        if (len > max) {
            max = len;
            strcpy(longest, line);
        }
    }

    if (max > 0) {  // 如果有行被读取，则存在最长行
        printf("The longest line is:\n%s", longest);
        printf("The length of the longest line is: %d\n", max);
    }

    return 0;
}
```