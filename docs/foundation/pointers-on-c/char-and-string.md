# 字符串、字符和字节
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

字符串是一种重要的数据类型，但是C语言并没有显式的字符串数据类型，因为字符串以字符串常量的形式出现或者存储于字符数组中。

## 字符串基础
首先，让我们回顾一下字符串的基础知识。字符串就是一串零个或多个字符，并且以一个位模式为全0的NUL字节结尾。因此，字符串所包含的字符内部不能出现NUL字节。这个限制很少会引起问题，因为NUL字节并不存在与它相关联的可打印字符，这也是它被选为终止符的原因。NUL字节是字符串的终止符，但它本身并不是字符串的一部分，所以字符串的长度并不包括NUL字节。

## 字符串查找基础
### 查找一个字符
在一个字符串中查找一个特定字符最容易的方法是使用strchr和strrchr函数

### 查找任何几个字符
strpbrk是个更为常见的函数。它并不是查找某个特定的字符，而是查找任何一组字符第1次在字符串中出现的位置

### 查找一个子串
为了在字符串中查找一个子串，我们可以使用strstr函数

## 高级字符串查找
### 查找一个字符串前缀
strspn和strcspn函数用于在字符串的起始位置对字符计数

### 查找标记
一个字符串常常包含几个单独的部分，它们彼此被分隔开来。每次为了处理这些部分，你首先必须把它们从字符串中抽取出来。

这个任务正是strtok函数所实现的功能。它从字符串中隔离各个单独的称为标记(token)的部分，并丢弃分隔符

## 内存操作
根据定义，字符串由一个NUL字节结尾，所以字符串内部不能包含任何NUL字符。但是，非字符串数据内部包含零值的情况并不罕见。你无法使用字符串函数来处理这种类型的数据，因为当它们遇到第1个NUL字节时将停止工作。

不过，我们可以使用另外一组相关的函数，它们的操作与字符串函数类似，但这些函数能够处理任意的字节序列。

## 总结
字符串就是零个或多个字符的序列，该序列以一个NUL字节结尾。字符串的长度就是它所包含的字符的数目。标准库提供了一些函数用于处理字符串，它们的原型位于头文件string.h中。

strlen函数用于计算一个字符串的长度，它的返回值是一个无符号整数，所以把它用于表达式时应该小心。strcpy函数把一个字符串从一个位置复制到另一个位置，而strcat函数把一个字符串的一份拷贝添加到另一个字符串的后面。这两个函数都假定它们的参数是有效的字符串，而且如果源字符串和目标字符串出现重叠，函数的结果是未定义的。strcmp对两个字符串进行词典序的比较。它的返回值提示第1个字符串是大于、小于还是等于第2个字符串。

长度受限的函数strncpy、strncat和strncmp都类似它们对应的不受限制版本。区别在于这些函数还接受一个长度参数。在strncpy中，长度指定了多少个字符将被写入到目标字符数组中。如果源字符串比指定长度更长，结果字符串将不会以NUL字节结尾。strncat函数的长度参数指定从源字符串复制过来的字符的最大数目，但它的结果始终以一个NUL字节结尾。strcmp函数的长度参数用于限定字符比较的数目。如果两个字符串在指定的数目里不存在区别，它们便被认为是相等的。

用于查找字符串的函数有好几个。strchr函数查找一个字符串中某个字符第1次出现的位置。strrchr函数查找一个字符串中某个字符最后一次出现的位置。strpbrk在一个字符串中查找一个指定字符集中任意字符第1次出现的位置。strstr函数在一个字符串中查找另一个字符串第1次出现的位置。

标准库还提供了一些更加高级的字符串查找函数。strspn函数计算一个字符串的起始部分匹配一个指定字符集中任意字符的字符数量。strcspn函数计算一个字符串的起始部分不匹配一个指定字符集中任意字符的字符数量。strtok函数把一个字符串分割成几个标记。每次当它调用时，都返回一个指向字符串中下一个标记位置的指针。这些标记由一个指定字符集的一个或多个字符分隔。

strerror把一个错误代码作为它的参数。它返回一个指向字符串的指针，该字符串用于描述这个错误。

标准库还提供了各种用于测试和转换字符的函数。使用这些函数的程序比那些自己执行字符测试和转换的程序更具移植性。toupper函数把一个小写字母字符转换为大写形式，tolower函数则执行相反的任务。iscntrl函数检查它的参数是不是一个控制字符，isspace函数测试它的参数是否为空白字符。isdigit函数用于测试它的参数是否为一个十进制数字字符，isxdigit函数则检查它的参数是否为一个十六进制数字字符。islower和isupper函数分别检查它们的参数是否为大写和小写字母。isalpha函数检查它的参数是否为字母字符，isalnum函数检查它的参数是否为字母或数字字符，ispunct函数检查它的参数是否为标点符号字符。最后，isgraph函数检查它的参数是否为图形字符，isprint函数检查它的参数是否为图形字符或空白字符。

memxxx函数提供了类似字符串函数的能力，但它们可以处理包括NUL字节在内的任意字节。这些函数都接受一个长度参数。memcpy从源参数向目标参数复制由长度参数指定的字节数。memmove函数执行相同的功能，但它能够正确处理源参数和目标参数出现重第9章　字符串、字符和字节
字符串是一种重要的数据类型，但是C语言并没有显式的字符串数据类型，因为字符串以字符串常量的形式出现或者存储于字符数组中。字符串常量很适用于那些程序不会对它们进行修改的字符串。所有其他字符串都必须存储于字符数组或动态分配的内存中（见第11章）。本章描述处理字符串和字符的库函数，以及一组相关的，具有类似能力的，既可以处理字符串也可以处理非字符串数据的函数。

## 编程练习
### 练习 1
编写一个程序，从标准输入读取一些字符，并统计下列各类字符所占的百分比。

控制字符
空白字符
数字
小写字母
大写字母
标点符号
不可打印的字符

```c
#include <stdio.h>
#include <ctype.h>

int main() {
    int control = 0, space = 0, digit = 0, lower = 0, upper = 0, punct = 0, nonprint = 0, total = 0;
    int ch;
    
    // 读取字符直到遇到EOF
    while ((ch = getchar()) != EOF) {
        total++;

        if (iscntrl(ch)) control++;
        if (isspace(ch)) space++;
        if (isdigit(ch)) digit++;
        if (islower(ch)) lower++;
        if (isupper(ch)) upper++;
        if (ispunct(ch)) punct++;
        if (ch > 127) nonprint++; // 非ASCII字符
    }
    
    // 打印统计结果
    printf("总字符数: %d\n", total);
    printf("控制字符: %.2f%%\n", (double)control/total*100);
    printf("空白字符: %.2f%%\n", (double)space/total*100);
    printf("数字: %.2f%%\n", (double)digit/total*100);
    printf("小写字母: %.2f%%\n", (double)lower/total*100);
    printf("大写字母: %.2f%%\n", (double)upper/total*100);
    printf("标点符号: %.2f%%\n", (double)punct/total*100);
    printf("不可打印的字符: %.2f%%\n", (double)nonprint/total*100);
    
    return 0;
}
```

### 练习 2
编写一个名叫my_strcpy的函数。它类似于strcpy函数，但它不会溢出目标数组。复制的结果必须是一个真正的字符串。

```c
#include <stdio.h>

// my_strcpy 函数的声明
void my_strcpy(char *dest, const char *src, size_t destSize);

int main() {
    char dest[10];
    const char *src = "HelloWorld";

    // 使用 my_strcpy 复制字符串，确保不发生溢出
    my_strcpy(dest, src, sizeof(dest));
    
    printf("Copied string: %s\n", dest);
    return 0;
}

// my_strcpy 函数的定义
void my_strcpy(char *dest, const char *src, size_t destSize) {
    if (destSize == 0) return; // 如果目标数组的大小为0，则无法进行复制
    
    size_t i;
    for (i = 0; i < destSize - 1 && src[i] != '\0'; i++) {
        dest[i] = src[i];
    }
    dest[i] = '\0'; // 确保字符串以空字符结尾
}
```

### 练习 3
编写一个名叫my_strcat的函数。它类似于strcat函数，但它不会溢出目标数组。它的结果必须是一个真正的字符串。

```c
#include <stdio.h>

// my_strcat 函数的声明
void my_strcat(char *dest, const char *src, size_t destSize);

int main() {
    char dest[20] = "Hello, ";
    const char *src = "World!";

    // 使用 my_strcat 连接两个字符串，确保不发生溢出
    my_strcat(dest, src, sizeof(dest));
    
    printf("Concatenated string: %s\n", dest);
    return 0;
}

// my_strcat 函数的定义
void my_strcat(char *dest, const char *src, size_t destSize) {
    size_t destLen = 0;
    while (destLen < destSize && dest[destLen] != '\0') {
        destLen++; // 计算现有字符串的长度（不包括结尾的'\0'）
    }

    if (destLen == destSize) {
        return; // 如果目标字符串已经占满了整个数组，则无法添加更多字符
    }

    // 复制源字符串到目标字符串的末端，直到源字符串结束或目标数组被填满
    size_t i;
    for (i = 0; destLen + i < destSize - 1 && src[i] != '\0'; i++) {
        dest[destLen + i] = src[i];
    }
    dest[destLen + i] = '\0'; // 确保字符串以'\0'结尾
}
```

### 练习 4
编写函数

```
int palindrome( char *string );
```

如果参数字符串是个回文，函数就返回真，否则就返回假。回文就是指一个字符串从左向右读和从右向左读是一样的 。函数应该忽略所有的非字母字符，而且在进行字符比较时不用区分大小写。

```c
#include <stdio.h>
#include <ctype.h> // 引入ctype.h以使用isalpha和tolower函数

// 函数声明
int palindrome(char *string);

int main() {
    char testString[] = "A man, a plan, a canal: Panama";
    
    if (palindrome(testString)) {
        printf("\"%s\" is a palindrome.\n", testString);
    } else {
        printf("\"%s\" is not a palindrome.\n", testString);
    }
    
    return 0;
}

// 判断字符串是否是回文的函数实现
int palindrome(char *string) {
    // 首先确定字符串的长度
    int length = 0;
    while (string[length] != '\0') {
        length++;
    }
    
    // 使用双指针技术，一个从前往后，一个从后往前
    int left = 0, right = length - 1;
    while (left < right) {
        // 如果左边字符不是字母，则左指针右移
        if (!isalpha((unsigned char)string[left])) {
            left++;
            continue;
        }
        // 如果右边字符不是字母，则右指针左移
        if (!isalpha((unsigned char)string[right])) {
            right--;
            continue;
        }
        // 比较两边的字符是否相等（不区分大小写）
        if (tolower((unsigned char)string[left]) != tolower((unsigned char)string[right])) {
            return 0; // 如果不相等，返回0
        }
        // 如果相等，继续比较
        left++;
        right--;
    }
    return 1; // 字符串是回文，返回1
}
```