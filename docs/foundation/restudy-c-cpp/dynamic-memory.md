# 动态内存分配
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

数组的元素存储于内存中连续的位置上。当一个数组被声明时，它所需要的内存在编译时就被分配。但是，你也可以使用动态内存分配在运行时为它分配内存。

## 为什么使用动态内存分配
当你声明数组时，你必须用一个编译时常量指定数组的长度。但是，数组的长度常常在运行时才知道，这是由于它所需要的内存空间取决于输入数据。例如，一个用于计算学生等级和平均分的程序可能需要存储一个班级所有学生的数据，但不同班级的学生数量可能不同。在这些情况下，我们通常采取的方法是声明一个较大的数组，它可以容纳可能出现的最多元素。

### malloc和free
C函数库提供了两个函数，malloc和free，分别用于执行动态内存分配和释放。这些函数维护一个可用内存池。当一个程序另外需要一些内存时，它就调用malloc函数，malloc从内存池中提取一块合适的内存，并向该程序返回一个指向这块内存的指针。这块内存此时并没有以任何方式进行初始化。如果对这块内存进行初始化非常重要，你要么自已动手对它进行初始化，要么使用calloc函数（在下一节描述）。当一块以前分配的内存不再使用时，程序调用free函数把它归还给内存池供以后之需。

malloc的参数就是需要分配的内存字节（字符）数 。如果内存池中的可用内存可以满足这个需求，malloc就返回一个指向被分配的内存块起始位置的指针。

malloc所分配的是一块连续的内存。例如，如果请求它分配100个字节的内存，那么它实际分配的内存就是100个连续的字节，并不会分开位于两块或多块不同的内存。同时，malloc实际分配的内存有可能比你请求的稍微多一点。但是，这个行为是由编译器定义的，所以你不能指望它肯定会分配比你的请求更多的内存。

如果内存池是空的，或者它的可用内存无法满足你的请求，会发生什么情况呢？在这种情况下，malloc函数向操作系统请求，要求得到更多的内存，并在这块新内存上执行分配任务。如果操作系统无法向malloc提供更多的内存，malloc就返回一个NULL指针。因此，对每个从malloc返回的指针都进行检查，确保它并非NULL是非常重要的。

free的参数必须要么是NULL，要么是一个先前从malloc、calloc或realloc返回的值。向free传递一个NULL参数不会产生任何效果。

malloc又是如何知道你所请求的内存需要存储的是整数、浮点值、结构还是数组呢？它并不知情——malloc返回一个类型为void *的指针，正是缘于这个原因。标准表示一个void *类型的指针可以转换为其他任何类型的指针。但是，有些编译器，尤其是那些老式的编译器，可能要求你在转换时使用强制类型转换。

对于要求边界对齐的机器，malloc所返回的内存的起始位置将始终能够满足对边界对齐要求最严格的类型的要求。

### calloc和realloc
另外还有两个内存分配函数，calloc和realloc。

```c
void　　*calloc( size_t num_elements,
　　　　　　　　　 size_t element_size );
void　　realloc( void *ptr, size_t new_size );
```

calloc也用于分配内存。malloc和calloc之间的主要区别是后者在返回指向内存的指针之前把它初始化为0。这个初始化常常能带来方便，但如果你的程序只是想把一些值存储到数组中，那么这个初始化过程纯属浪费时间。calloc和malloc之间另一个较小的区别是它们请求内存数量的方式不同。calloc的参数包括所需元素的数量和每个元素的字节数。根据这些值，它能够计算出总共需要分配的内存。

realloc函数用于修改一个原先已经分配的内存块的大小。使用这个函数，你可以使一块内存扩大或缩小。如果它用于扩大一个内存块，那么这块内存原先的内容依然保留，新增加的内存添加到原先内存块的后面，新内存并未以任何方法进行初始化。如果它用于缩小一个内存块，该内存块尾部的部分内存便被拿掉，剩余部分内存的原先内容依然保留。

如果原先的内存块无法改变大小，realloc将分配另一块正确大小的内存，并把原先那块内存的内容复制到新的块上。因此，在使用realloc之后，你就不能再使用指向旧内存的指针，而是应该改用realloc所返回的新指针。

最后，如果realloc函数的第1个参数是NULL，那么它的行为就和malloc一模一样。

## 使用动态分配的内存
当你使用动态分配内存后就有了一个指针，那么你该如何使用这块内存呢？当然，你可以使用间接访问和指针运算来访问数组的不同整数位置，同时你不仅可以使用指针，也可以使用下标。

## 常见的动态内存错误
在使用动态内存分配的程序中，常常会出现许多错误。这些错误包括对NULL指针进行解引用操作、对分配的内存进行操作时越过边界、释放并非动态分配的内存、试图释放一块动态分配的内存的一部分以及一块动态内存被释放之后被继续使用。

当一个使用动态内存分配的程序失败时，人们很容易把问题的责任推给malloc和free函数。但它们实际上很少是罪魁祸首。事实上，问题几乎总是出在你自己的程序中，而且常常是由于访问了分配内存以外的区域而引起的。

## 总结
当数组被声明时，必须在编译时知道它的长度。动态内存分配允许程序为一个长度在运行时才知道的数组分配内存空间。

malloc和calloc函数都用于动态分配一块内存，并返回一个指向该块内存的指针。malloc的参数就是需要分配的内存的字节数。和它不同的是，calloc的参数是你需要分配的元素个数和每个元素的长度。calloc函数在返回前把内存初始化为零，而malloc函数返回时内存并未以任何方式进行初始化。调用realloc函数可以改变一块已经动态分配的内存的大小。增加内存块大小时有可能采取的方法是把原来内存块上的所有数据复制到一个新的、更大的内存块上。当一个动态分配的内存块不再使用时，应该调用free函数把它归还给可用内存池。内存被释放之后便不能再被访问。

如果请求的内存分配失败，malloc、calloc和realloc函数返回的将是一个NULL指针。错误地访问分配内存之外的区域所引起的后果类似越界访问一个数组，但这个错误还可能破坏可用内存池，导致程序失败。如果一个指针不是从早先的malloc、calloc或realloc函数返回的，它是不能作为参数传递给free函数的。你也不能只释放一块内存的一部分。

内存泄漏是指内存被动态分配以后，当它不再使用时未被释放。内存泄漏会增加程序的体积，有可能导致程序或系统的崩溃。

## 编程练习
### 练习 1
编写一个函数，从标准输入读取一列整数，把这些值存储于一个动态分配的数组中并返回这个数组。函数通过观察EOF判断输入列表是否结束。数组的第1个数是数组包含的值的个数，它的后面就是这些整数值。

```c
#include <stdio.h>
#include <stdlib.h>

int* readIntegers() {
    int* array = (int*)malloc(sizeof(int)); // 初始分配空间给计数器
    if (array == NULL) { // 检查内存分配是否成功
        perror("Initial malloc failed");
        exit(EXIT_FAILURE);
    }
    array[0] = 0; // 初始化计数器为0

    int input;
    while (scanf("%d", &input) != EOF) { // 读取整数直到EOF
        array[0]++; // 增加计数器
        // 重新分配空间以容纳新的整数和计数器
        int* temp = (int*)realloc(array, (array[0] + 1) * sizeof(int));
        if (temp == NULL) { // 检查内存重新分配是否成功
            perror("realloc failed");
            free(array);
            exit(EXIT_FAILURE);
        }
        array = temp;
        array[array[0]] = input; // 在数组的末尾添加新读取的整数
    }
    
    if (array[0] == 0) { // 如果没有读取任何整数
        free(array);
        return NULL;
    }

    return array; // 返回动态分配并填充的数组
}

int main() {
    int* integers = readIntegers();
    if (integers != NULL) {
        printf("You have entered %d integers which are:\n", integers[0]);
        for (int i = 1; i <= integers[0]; i++) {
            printf("%d\n", integers[i]);
        }
        free(integers); // 记得释放分配的内存
    } else {
        printf("No integers were read.\n");
    }

    return 0;
}
```

### 练习 2
编写一个函数，从标准输入读取一个字符串，把字符串复制到动态分配的内存中，并返回该字符串的拷贝。这个函数不应该对读入字符串的长度作任何限制！

```c
#include <stdio.h>
#include <stdlib.h>

char* readAndCopyString() {
    int bufferSize = 10; // 初始缓冲区大小
    char* str = (char*)malloc(bufferSize * sizeof(char));
    if (str == NULL) { // 检查内存分配是否成功
        perror("Initial malloc failed");
        exit(EXIT_FAILURE);
    }

    char ch;
    int count = 0;
    printf("Enter a string: ");
    while (scanf("%c", &ch) && ch != '\n') { // 读取字符直到遇到换行符
        str[count++] = ch;
        if (count == bufferSize) { // 若当前已用完缓冲区
            bufferSize *= 2; // 增加缓冲区大小
            char* temp = (char*)realloc(str, bufferSize * sizeof(char));
            if (temp == NULL) { // 检查内存重新分配是否成功
                perror("realloc failed");
                free(str);
                exit(EXIT_FAILURE);
            }
            str = temp;
        }
    }
    str[count] = '\0'; // 在字符串末尾添加NULL终止符

    // 重新分配恰好合适的空间给字符串
    char* exactStr = (char*)realloc(str, (count + 1) * sizeof(char));
    if (exactStr == NULL) { // 检查内存重新分配是否成功
        perror("Final realloc failed");
        free(str); // 释放原来的内存防止内存泄露
        exit(EXIT_FAILURE);
    }

    return exactStr; // 返回动态分配的字符串副本
}

int main() {
    char* str = readAndCopyString();
    printf("You entered: %s\n", str);
    free(str); // 使用完毕后释放分配的内存
    return 0;
}
```

















