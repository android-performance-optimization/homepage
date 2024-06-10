# 高级指针话题
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

## 函数指针
你不会每天都使用函数指针。但是，它们确有用武之地，最常见的两个用途是转换表(jump table)和作为参数传递给另一个函数。

```c
int　　f( int );
int　　(*pf)( int ) = &f;
```

初始化表达式中的&操作符是可选的，因为函数名 被使用时总是由编译器把它转换为函数指针 。&操作符只是显式地说明了编译器将隐式执行的任务。

## 命令行参数
处理命令行参数是指向指针的指针的另一个用武之地。有些操作系统，包括UNIX和MS-DOS，让用户在命令行中编写参数来启动一个程序的执行。这些参数被传递给程序，程序按照它认为合适的任何方式对它们进行处理。

## 总结
如果声明得当，一个指针变量可以指向另一个指针变量。和其他的指针变量一样，一个指向指针的指针在它使用之前必须进行初始化。为了取得目标对象，必须对指针的指针执行双重的间接访问操作。更多层的间接访问也是允许的（比如一个指向整型的指针的指针的指针），但它们与简单的指针相比用的较少。你也可以创建指向函数和数组的指针，还可以创建包含这类指针的数组。

你可以使用函数指针来实现回调函数。一个指向回调函数的指针作为参数传递给另一个函数，后者使用这个指针调用回调函数。使用这种技巧，你可以创建通用型函数，用于执行普通的操作如在一个链表中查找。任何特定问题的某个实例的工作，如在链表中进行值的比较，由客户提供的回调函数执行。

转移表也使用函数指针。转移表像switch语句一样执行选择。转移表由一个函数指针数组组成（这些函数必须具有相同的原型）。函数通过下标选择某个指针，再通过指针调用对应的函数。你必须始终保证下标值处于适当的范围之内，因为在转移表中调试错误是非常困难的。

如果某个执行环境实现了命令行参数，这些参数是通过两个形参传递给main函数的。这两个形参通常称为argc和argv。argc是一个整数，用于表示参数的数量。argv是一个指针，它指向一个序列的字符型指针。该序列中的每个指针指向一个命令行参数。该序列以一个NULL指针作为结束标志。其中第1个参数就是程序的名字。程序可以通过对argv使用间接访问操作来访问命令行参数。

出现在表达式中的字符串常量的值是一个常量指针，它指向字符串的第1个字符。和数组名一样，你既可以用指针表达式也可以用下标来使用字符串常量。

## 编程练习
### 练习 1
编写一个通用目的的函数，遍历一个单链表。它应该接受两个参数：一个指向链表第1个节点的指针和一个指向一个回调函数的指针。回调函数应该接受单个参数，也就是指向一个链表节点的指针。对于链表中的每个节点，都应该调用一次这个回调函数。这个函数需要知道链表节点的什么信息？

```c
typedef void (*callback_fn)(NODE *);

void traverse_list(NODE *head, callback_fn callback) {
    NODE *current = head;  // 从链表头开始遍历
    
    while (current != NULL) {
        callback(current);  // 对当前节点调用回调函数
        current = current->next;  // 移动到下一个节点
    }
}
```

### 练习 2
转换下面的代码段，使它改用转移表而不是switch语句

```c
void add_new_trans(Node *list, Transaction *transaction);
void delete_trans(Node *list, Transaction *transaction);
void forward_trans(Node *list, Transaction *transaction);
void backward_trans(Node *list, Transaction *transaction);
void search_trans(Node *list, Transaction *transaction);
void edit_trans(Node *list, Transaction *transaction);

void (*trans_table[])(Node *, Transaction *) = {
    add_new_trans,
    delete_trans,
    forward_trans,
    backward_trans,
    search_trans,
    edit_trans
};

// 假设transaction->type是Trans_type枚举中的一个有效值
trans_table[transaction->type](list, transaction);
```

### 练习 3
编写一个名叫sort的函数，它用于对一个任何类型的数组进行排序。为了使函数更为通用，它的其中一个参数必须是一个指向比较回调函数的指针，该回调函数由调用程序提供。比较函数接受两个参数，也就是两个指向需要进行比较的值的指针。如果两个值相等，函数返回零；如果第1个值小于第2个，函数返回一个小于零的整数；如果第1个值大于第2个，函数返回一个大于零的整数。

```c
// 比较回调函数的类型定义
typedef int (*compare_fn)(const void *, const void *);

#include <stdlib.h>

// 通用的排序函数
void sort(void *array, size_t nitems, size_t size, compare_fn cmp) {
    // 用一个临时变量来保存元素，以进行交换
    void *temp = malloc(size);
    if (temp == NULL) {
        // 处理内存分配失败的情况
        return;
    }

    for (size_t i = 0; i < nitems - 1; ++i) {
        for (size_t j = 0; j < nitems - i - 1; ++j) {
            // 计算数组中当前元素和下一个元素的指针
            void *current = (char*)array + j * size;
            void *next = (char*)array + (j + 1) * size;
            
            // 使用提供的比较函数进行比较
            if (cmp(current, next) > 0) {
                // 如果current > next，则交换它们
                memcpy(temp, current, size);
                memcpy(current, next, size);
                memcpy(next, temp, size);
            }
        }
    }

    free(temp); // 释放用于交换的临时内存
}

#include <stdio.h>

// 整数比较函数
int compare_ints(const void *a, const void *b) {
    return (*(int*)a - *(int*)b);
}

int main() {
    int arr[] = {5, 3, 2, 4, 1};
    size_t n = sizeof(arr) / sizeof(arr[0]);

    sort(arr, n, sizeof(arr[0]), compare_ints);

    // 打印排序后的数组
    for (size_t i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");

    return 0;
}
```


