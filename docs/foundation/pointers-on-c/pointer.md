# 指针
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

## 内存和地址
前面提到过，我们可以把计算机的内存看作是一条长街上的一排房屋。每座房子都可以容纳数据，并通过一个房号来标识。

这个比喻颇为有用，但也存在局限性。计算机的内存由数以亿万计的位（bit）组成，每个位可以容纳值0或1。由于一个位所能表示的值的范围太有限，所以单独的位用处不大，通常许多位合成一组作为一个单位，这样就可以存储范围较大的值。

1．内存中的每个位置由一个独一无二的地址标识。

2．内存中的每个位置都包含一个值。

我们可以通过内存地址获得这个地址上的值，但是，要记住所有这些地址实在是太笨拙了，所以高级语言所提供的特性之一就是通过名字而不是地址来访问内存的位置。

当然，这些名字就是我们所称的变量。有一点非常重要，你必须记住，名字与内存位置之间的关联并不是硬件所提供的，它是由编译器为我们实现的。所有这些变量给了我们一种更方便的方法记住地址——硬件仍然通过地址访问内存位置 。

## 间接访问操作符
通过一个指针访问它所指向的地址的过程称为间接访问(indirection)或解引用指针(dereferencing the pointer)。这个用于执行间接访问的操作符是单目操作符*。

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2024/may/p4.png)


## NULL指针
标准定义了NULL指针，它作为一个特殊的指针变量，表示不指向任何东西。要使一个指针变量为NULL，你可以给它赋一个零值。为了测试一个指针变量是否为NULL，你可以将它与零值进行比较。之所以选择零这个值是因为一种源代码约定。就机器内部而言，NULL指针的实际值可能与此不同。在这种情况下，编译器将负责零值和内部值之间的翻译转换。

NULL指针的概念是非常有用的，因为它给了你一种方法，表示某个特定的指针目前并未指向任何东西。例如，一个用于在某个数组中查找某个特定值的函数可能返回一个指向查找到的数组元素的指针。如果该数组不包含指定条件的值，函数就返回一个NULL指针。这个技巧允许返回值传达两个不同片段的信息。首先，有没有找到元素？其次，如果找到，它是哪个元素？

对指针进行解引用操作可以获得它所指向的值。但从定义上看，NULL指针并未指向任何东西。因此，对一个NULL指针进行解引用操作是非法的。在对指针进行解引用操作之前，你首先必须确保它并非NULL指针。

## 指针、间接访问和变量
如果你自以为已经精通了指针，请看一下这个表达式，看看你是否明白它的意思。

```c
*&a = 25;
```

如果你的答案是它把值25赋值给变量a，恭喜！你答对了。让我们来分析这个表达式。首先，&操作符产生变量a的地址，它是一个指针常量（注意，使用这个指针常量并不需要知道它的实际值）。接着，*操作符访问其操作数所表示的地址。在这个表达式中，操作数是a的地址，所以值25就存储于a中。

这条语句和简单地使用a=25;有什么区别吗？从功能上说，它们是相同的。但是，它涉及更多的操作。除非编译器（或优化器）知道你在干什么并丢弃额外的操作，否则它所产生的目标代码将会更大、更慢。更糟的是，这些额外的操作符会使源代码的可读性变差。基于这些原因，没人会故意使用像*&a这样的表达式。

## 指针运算
指针加上一个整数的结果是另一个指针。问题是，它指向哪里？如果你将一个字符指针加1，运算结果产生的指针指向内存中的下一个字符。float占据的内存空间不止1个字节，如果你将一个指向float的指针加1，将会发生什么呢？它会不会指向该float值内部的某个字节呢？

幸运的是，答案是否定的。当一个指针和一个整数量执行算术运算时，整数在执行加法运算前始终会根据合适的大小进行调整。这个“合适的大小”就是指针所指向类型的大小，“调整”就是把整数值和“合适的大小”相乘。为了更好地说明，试想在某台机器上，float占据4个字节。在计算float型指针加3的表达式时，这个3将根据float类型的大小（此例中为4）进行调整（相乘）。这样，实际加到指针上的整型值为12。

### 算术运算
#### 指针　±　整数

标准定义这种形式只能用于指向数组中某个元素的指针，但实际上这种形式也适用于使用 malloc 函数动态分配获得的内存，尽管翻遍标准也未见它提及这个事实。

数组中的元素存储于连续的内存位置中，后面元素的地址大于前面元素的地址。因此，我们很容易看出，对一个指针加1使它指向数组中下一个元素。

第2种类型的指针运算具有如下形式：

#### 指针　—　指针
只有当两个指针都指向同一个数组中的元素时，才允许从一个指针减去另一个指针。

两个指针相减的结果的类型是ptrdiff_t，它是一种有符号整数类型。减法运算的值是两个指针在内存中的距离（以数组元素的长度为单位，而不是以字节为单位），因为减法运算的结果将除以数组元素类型的长度。例如，如果p1指向array[i]而p2指向array[j]，那么p2-p1的值就是j-i的值。

如果两个指针所指向的不是同一个数组中的元素，那么它们之间相减的结果是未定义的。就像如果你把两个位于不同街道的房子的门牌号码相减不可能获得这两所房子间的房子数一样。程序员无从知道两个数组在内存中的相对位置，如果不知道这一点，两个指针之间的距离就毫无意义。

### 关系运算
对指针执行关系运算也是有限制的。用下列关系操作符对两个指针值进行比较是可能的：

```
<　　<=　　>　　>=
```

不过前提是它们都指向同一个数组中的元素。根据你所使用的操作符，比较表达式将告诉你哪个指针指向数组中更前或更后的元素。标准并未定义如果两个任意的指针进行比较会产生什么结果。

然而，你可以在两个任意的指针间执行相等或不相等测试，因为这类比较的结果和编译器选择在何处存储数据并无关系——指针要么指向同一个地址，要么指向不同的地址。

## 总结
计算机内存中的每个位置都由一个地址标识。通常，邻近的内存位置合成一组，这样就允许存储更大范围的值。指针就是它的值表示内存地址的变量。

无论是程序员还是计算机都无法通过值的位模式来判断它的类型。类型是通过值的使用方法隐式地确定的。编译器能够保证值的声明和值的使用之间的关系是适当的，从而帮助我们确定值的类型。

指针变量的值并非它所指向的内存位置所存储的值。我们必须使用间接访问来获得它所指向位置存储的值。对一个“指向整型的指针”施加间接访问操作的结果将是一个整型值。

声明一个指针变量并不会自动分配任何内存。在对指针执行间接访问前，指针必须进行初始化：或者使它指向现有的内存，或者给它分配动态内存。对未初始化的指针变量执行间接访问操作是非法的，而且这种错误常常难以检测。其结果常常是一个不相关的值被修改。这种错误是很难被调试发现的。

NULL指针就是不指向任何东西的指针。它可以赋值给一个指针，用于表示那个指针并不指向任何值。对NULL指针执行间接访问操作的后果因编译器而异，两个常见的后果分别是返回内存位置零的值以及终止程序。

和任何其他变量一样，指针变量也可以作为左值使用。对指针执行间接访问操作所产生的值也是个左值，因为这种表达式标识了一个特定的内存位置。

除了NULL指针之外，再也没有任何内建的记法来表示指针常量，因为程序员通常无法预测编译器会把变量放在内存中的什么位置。在极少见的情况下，我们偶尔需要使用指针常量，这时我们可以通过把一个整型值强制转换为指针类型来创建它。

在指针值上可以执行一些有限的算术运算。你可以把一个整型值加到一个指针上，也可以从一个指针减去一个整型值。在这两种情况下，这个整型值会进行调整，原值将乘以指针目标类型的长度。这样，对一个指针加1将使它指向下一个变量，至于该变量在内存中占几个字节的大小则与此无关。

然而，指针运算只有作用于数组中其结果才是可以预测的。对任何并非指向数组元素的指针执行算术运算是非法的（但常常很难被检测到）。如果一个指针减去一个整数后，运算结果产生的指针所指向的位置在数组第一个元素之前，那么它也是非法的。加法运算稍有不同，如果结果指针指向数组最后一个元素后面的那个内存位置仍是合法（但不能对这个指针执行间接访问操作），不过再往后就不合法了。

如果两个指针都指向同一个数组中的元素，那么它们之间可以相减。指针减法的结果经过调整（除以数组元素类型的长度），表示两个指针在数组中相隔多少个元素。如果两个指针并不是指向同一个数组的元素，那么它们之间进行相减就是错误的。

任何指针之间都可以进行比较，测试它们相等或不相等。如果两个指针都指向同一个数组中的元素，那么它们之间还可以执行<、<=、>和>=等关系运算，用于判断它们在数组中任何指针之间都可以进行比较，测试它们相等或不相等。如果两个指针都指向同一个数组中的元素，那么它们之间还可以执行<、<=、>和>=等关系运算，用于判断它们在数组中的相对位置。对两个不相关的指针执行关系运算，其结果是未定义的。

## 编译练习
### 练习 1
请编写一个函数，它在一个字符串中进行搜索，查找所有在一个给定字符集合中出现的字符。这个函数的原型应该如下：

```c
char *find_char( char const *source,
　　char const *chars );
```

```c
#include <stdio.h>

char *find_char(char const *source, char const *chars) {
    if (!source || !chars) {
        return NULL; // 如果任一输入字符串是NULL，直接返回NULL
    }

    // 遍历source字符串
    for (; *source != '\0'; ++source) {
        // 对每个source中的字符，遍历chars字符串
        for (const char *temp_chars = chars; *temp_chars != '\0'; ++temp_chars) {
            // 如果在chars中找到了与source中当前字符相匹配的字符，返回当前字符的地址
            if (*source == *temp_chars) {
                return (char *)source;
            }
        }
    }

    // 如果遍历完了source字符串也没有在chars中找到匹配的字符，返回NULL
    return NULL;
}

int main() {
    char const *source = "Hello, World!";
    char const *chars = "abcdefghijklmnopqrstuvwxyz";

    char *found_char = find_char(source, chars);
    if (found_char != NULL) {
        printf("First matching character: '%c'\n", *found_char);
    } else {
        printf("No matching character found.\n");
    }

    return 0;
}

```
### 练习 2
请编写一个函数，删除一个字符串的一部分。函数的原型如下：

```
　int del_substr( char *str, char const *substr )
```

函数首先应该判断substr是否出现在str中。如果它并未出现，函数就返回0；如果出现，函数应该把str中位于该子串后面的所有字符复制到该子串的位置，从而删除这个子串，然后函数返回1。如果substr多次出现在str中，函数只删除第1次出现的子串。函数的第2个参数绝不会被修改。

```c
#include <stdio.h>
#include <string.h>
#include <stdbool.h>

// 功能: 查找子字符串substr在字符串str中的位置
// 返回值: 如果找到substr，则返回substr在str中首次出现的地址；否则返回NULL
char *find_substr(char *str, char const *substr) {
    if (!str || !substr) {
        return NULL;
    }

    int len_str = strlen(str);
    int len_substr = strlen(substr);

    if (len_substr == 0) {
        return str;
    }

    for (int i = 0; i <= len_str - len_substr; ++i) {
        bool found = true;
        for (int j = 0; j < len_substr; ++j) {
            if (str[i + j] != substr[j]) {
                found = false;
                break;
            }
        }
        if (found) {
            return &str[i];
        }
    }
    return NULL;
}

// 功能: 从字符串str中删除子字符串substr
// 返回值: 如果删除成功，返回1；如果子字符串未找到，返回0
int del_substr(char *str, char const *substr) {
    char *substr_location = find_substr(str, substr);

    if (substr_location == NULL) {
        // 如果未找到子字符串，返回0
        return 0;
    }

    int len_substr = strlen(substr);
    int shift = strlen(substr_location + len_substr) + 1; // 包括NULL终止符

    // 删除子字符串：将后面的字符串向前移动
    memmove(substr_location, substr_location + len_substr, shift);

    return 1;
}

int main() {
    char str[100] = "Hello, World! World!";
    char const *substr = "World";

    printf("Original string: '%s'\n", str);

    if (del_substr(str, substr)) {
        printf("After deletion: '%s'\n", str);
    } else {
        printf("Substring not found.\n");
    }

    return 0;
}
```

### 练习 3
编写函数reverse_string，它的原型如下：

```c
　void reverse_string( char *string );
```

函数把参数字符串中的字符反向排列。请使用指针而不是数组下标，不要使用任何C函数库中用于操纵字符串的函数。提示： 不需要声明一个局部数组来临时存储参数字符串。

```c
void reverse_string(char *string) {
    char *start = string; // start指针指向字符串开头
    char *end = string; // end指针也先指向开头，稍后寻找字符串末尾
    char temp; // 交换字符时使用的临时变量

    // 将end指针移动到字符串的末尾（不包括NULL终止符）
    while (*end != '\0') {
        end++;
    }
    end--; // end指针现在指向字符串的最后一个字符

    // 当start指针在end指针之前时，交换两个指针指向的字符
    while (start < end) {
        // 交换并移动指针
        temp = *start;
        *start = *end;
        *end = temp;

        // start指针向后移动，end指针向前移动
        start++;
        end--;
    }
}

int main() {
    char str[] = "Hello, World!";
    reverse_string(str);
    printf("%s\n", str); // 输出反转后的字符串
    return 0;
}

```














