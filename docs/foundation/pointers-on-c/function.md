# 函数
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

## 函数定义
函数的定义就是函数体的实现。函数体就是一个代码块，它在函数被调用时执行。与函数定义相反，函数声明出现在函数被调用的地方。函数声明向编译器提供该函数的相关信息，用于确保函数被正确地调用。

```c
function_name()
{
}
```

## 函数的参数
C函数的所有参数均以“传值调用”方式进行传递，这意味着函数将获得参数值的一份拷贝。这样，函数可以放心修改这个拷贝值，而不必担心会修改调用程序实际传递给它的参数。这个行为与Modula和Pascal中的值参数（不是var参数）相同。

C的规则很简单：所有参数都是传值调用。但是，如果被传递的参数是一个数组名，并且在函数中使用下标引用该数组的参数，那么在函数中对数组元素进行修改实际上修改的是调用程序中的数组元素。函数将访问调用程序的数组元素，数组并不会被复制。这个行为被称为“传址调用”，也就是许多其他语言所实现的var参数。

数组参数的这种行为似乎与传值调用规则相悖。但是，此处其实并无矛盾之处——数组名的值实际上是一个指针，传递给函数的就是这个指针的一份拷贝。下标引用实际上是间接访问的另一种形式，它可以对指针执行间接访问操作，访问指针指向的内存位置。参数（指针）实际上是一份拷贝，但在这份拷贝上执行间接访问操作所访问的是原先的数组。我们只要记住两个规则：

1．传递给函数的标量参数是传值调用的。

2．传递给函数的数组参数在行为上就像它们是通过传址调用的那样。

## 可变参数列表
在函数的原型中，列出了函数期望接受的参数，但原型只能显示固定数目的参数。让一个函数在不同的时候接受不同数目的参数是不是可以呢？

### stdarg宏
可变参数列表是通过宏来实现的，这些宏定义于stdarg.h头文件，它是标准库的一部分。这个头文件声明了一个类型va_list和三个宏——va_start、va_arg和va_end  。我们可以声明一个类型为va_list的变量，与这几个宏配合使用，访问参数的值。

你可能同时注意到参数列表中至少要有一个命名参数。如果连一个命名参数也没有，你就无法使用va_start。这个参数提供了一种方法，用于查找参数列表的可变部分。

对于这些宏，存在两个基本的限制。一个值的类型无法简单地通过检查它的位模式来判断，这两个限制就是这个事实的直接结果。

- 这些宏无法判断实际存在的参数的数量。
- 这些宏无法判断每个参数的类型。

## 编程练习
### 练习 1
为下面这个函数原型编写函数定义

```c
　　int ascii_to_integer( char *string );
```

这个字符串参数必须包含一个或多个数字，函数应该把这些数字字符转换为整数并返回这个整数。如果字符串参数包含了任何非数字字符，函数就返回零。

```c
#include <stdio.h>

int ascii_to_integer(char *string) {
    int result = 0;  // 存储结果的整数
    
    // 循环遍历字符串中的每个字符
    while (*string) {
        // 如果当前字符是数字字符
        if (*string >= '0' && *string <= '9') {
            // 将当前字符转换为整数（减去'0'的ASCII值得到真正的数值）
            // 并将其加到结果上，同时考虑当前结果已是前一次迭代的结果*10
            result = result * 10 + (*string - '0');
        } else {
            // 如果遇到非数字字符，则停止转换
            break;
        }
        
        // 移动到下一个字符
        string++;
    }
    
    return result;
}

// 测试函数的简单例子
int main() {
    char str1[] = "12345";
    char str2[] = "987a654";
    char str3[] = "abc123";
    
    printf("%s 转换为整数是: %d\n", str1, ascii_to_integer(str1));
    printf("%s 转换为整数是: %d\n", str2, ascii_to_integer(str2));
    printf("%s 转换为整数是: %d\n", str3, ascii_to_integer(str3));
    
    return 0;
}
```

### 练习 2
编写一个名叫max_list的函数，它用于检查任意数目的整型参数并返回它们中的最大值。参数列表必须以一个负值结尾，提示列表的结束。

```c
#include <stdio.h>
#include <stdarg.h>

// 定义max_list函数，使用可变参数列表找出并返回最大整数
int max_list(int first_arg, ...) {
    // 初始化可变参数列表
    va_list var_arg;
    va_start(var_arg, first_arg);

    // 假设第一个参数是目前为止的最大值
    int max_val = first_arg;
    
    int curr_val;

    // 循环直到遇到负数参数，这表示参数列表的结束
    while ((curr_val = va_arg(var_arg, int)) >= 0) {
        // 更新最大值
        if (curr_val > max_val) {
            max_val = curr_val;
        }
    }

    // 清理可变参数列表
    va_end(var_arg);

    // 返回找到的最大值
    return max_val;
}

// 主函数，用于展示max_list函数的使用
int main() {
    // 测试max_list函数
    printf("Max of 12, 75, 3, 87, -1 is: %d\n", max_list(12, 75, 3, 87, -1));
    printf("Max of 20, 10, 5, -1 is: %d\n", max_list(20, 10, 5, -1));
    printf("Max of 99, -1 is: %d\n", max_list(99, -1));

    return 0;
}
```

### 练习 3
实现一个简化的 printf 函数，它能够处理 %d、%f、%s和%c 格式码。根据 ANSI 标准的原则，其他格式码的行为是未定义的。你可以假定已经存在函数print_integer和print_float，用于打印这些类型的值。对于另外两种类型的值，使用putchar来打印。

```c
#include <stdio.h>
#include <stdarg.h>

// 假设的函数原型
void print_integer(int value) {
    printf("%d", value);
}

// 假设的函数原型
void print_float(float value) {
    printf("%f", value);
}

void mini_printf(const char *format, ...) {
    va_list args;
    va_start(args, format);

    while (*format != '\0') {
        if (*format == '%') {
            format++; // 移动到格式说明符
            switch (*format) {
                case 'd':
                    print_integer(va_arg(args, int));
                    break;
                case 'f':
                    print_float((float)va_arg(args, double)); // 注意：va_arg使用double来获取浮点数
                    break;
                case 's': ;
                    char *s = va_arg(args, char*);
                    while (*s) putchar(*s++);
                    break;
                case 'c':
                    putchar((char)va_arg(args, int)); // 注意：va_arg使用int来获取字符
                    break;
                default:
                    // 如果遇到未定义的格式代码，则输出它。
                    putchar('%');
                    putchar(*format);
                    break;
            }
        } else {
            // 如果不是格式指定符，直接输出该字符
            putchar(*format);
        }
        format++;
    }

    va_end(args);
}

int main() {
    mini_printf("This is a string: %s\n", "Hello, World!");
    mini_printf("This is an integer: %d\n", 12345);
    mini_printf("This is a float: %f\nThis is a char: %c\n", 3.14159, 'A');
    return 0;
}
```







