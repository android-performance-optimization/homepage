# 数组
本文为《C 与 指针读书笔记》，感兴趣的读者可以去看原书。

## 一维数组
### 数组名
```
int　a;
int　b[10];
```

我们把变量a称为标量，因为它是个单一的值，这个变量的类型是一个整数。我们把变量b称为数组，因为它是一些值的集合。下标和数组名一起使用，用于标识该集合中某个特定的值。例如，b[0]表示数组b的第1个值，b[4]表示第5个值。每个特定值都是一个标量，可以用于任何可以使用标量数据的上下文环境中。

b[4]的类型是整型，但b的类型又是什么？它所表示的又是什么？一个合乎逻辑的答案是它表示整个数组，但事实并非如此。在C中，在几乎所有使用数组名的表达式中，数组名的值是一个指针常量，也就是数组第1个元素的地址。它的类型取决于数组元素的类型：如果它们是int类型，那么数组名的类型就是“指向int的常量指针”；如果它们是其他类型 ，那么数组名的类型就是“指向其他类型 的常量指针”。

请不要根据这个事实得出数组和指针是相同的结论。数组具有一些和指针完全不同的特征。例如，数组具有确定数量的元素，而指针只是一个标量值。编译器用数组名来记住这些属性。只有当数组名在表达式中使用时，编译器才会为它产生一个指针常量。

注意这个值是指针常量，而不是指针变量。你不能修改常量的值。你只要稍微回想一下，就会认为这个限制是合理的：指针常量所指向的是内存中数组的起始位置，如果修改这个指针常量，唯一可行的操作就是把整个数组移动到内存的其他位置。但是，在程序完成链接之后，内存中数组的位置是固定的，所以当程序运行时，再想移动数组就为时已晚了。因此，数组名的值是一个指针常量。

只有在两种场合下，数组名并不用指针常量来表示——就是当数组名作为sizeof操作符或单目操作符&的操作数时。sizeof返回整个数组的长度，而不是指向数组的指针的长度。取一个数组名的地址所产生的是一个指向数组的指针，而不是一个指向某个指针常量值的指针。

你不能使用赋值符把一个数组的所有元素复制到另一个数组。你必须使用一个循环，每次复制一个元素。 

### 指针与下标
如果你可以互换地使用指针表达式和下标表达式，那么你应该使用哪一个呢？和往常一样，这里并没有一个简明答案。对于绝大多数人而言，下标更容易理解，尤其是在多维数组中。所以，在可读性方面，下标有一定的优势。但在另一方面，这个选择可能会影响运行时效率。

假定这两种方法都是正确的，下标绝不会比指针更有效率，但指针有时会比下标更有效率。

### 数组和指针
指针和数组并不是相等的。

```c
int  a[5];
int   *b;
```

a和b能够互换使用吗？它们都具有指针值，它们都可以进行间接访问和下标引用操作。但是，它们还是存在相当大的区别。

声明一个数组时，编译器将根据声明所指定的元素数量为数组保留内存空间，然后再创建数组名，它的值是一个常量，指向这段空间的起始位置。声明一个指针变量时，编译器只为指针本身保留内存空间，它并不为任何整型值分配内存空间。而且，指针变量并未被初始化为指向任何现有的内存空间，如果它是一个自动变量，它甚至根本不会被初始化。

因此，上述声明之后，表达式*a是完全合法的，但表达式*b却是非法的。*b将访问内存中某个不确定的位置，或者导致程序终止。另一方面，表达式b++可以通过编译，但a++却不行，因为a的值是个常量。

### 作为函数参数的数组名
当一个数组名作为参数传递给一个函数时会发生什么情况呢？你现在已经知道数组名的值就是一个指向数组第1个元素的指针，所以很容易明白此时传递给函数的是一份该指针的拷贝。函数如果执行了下标引用，实际上是对这个指针执行间接访问操作，并且通过这种间接访问，函数可以访问和修改调用程序的数组元素。

现在我可以解释C关于参数传递的表面上的矛盾之处。我早先曾说过所有传递给函数的参数都是通过传值方式进行的，但数组名参数的行为却仿佛它是通过传址调用传递的。传址调用是通过传递一个指向所需元素的指针，然后在函数中对该指针执行间接访问操作实现对数据的访问。作为参数的数组名是个指针，下标引用实际执行的就是间接访问。

那么数组的传值调用行为又是表现在什么地方呢？传递给函数的是参数的一份拷贝（指向数组起始位置的指针的拷贝），所以函数可以自由地操作它的指针形参，而不必担心会修改对应的作为实参的指针。

所以，此处并不存在矛盾：所有的参数都是通过传值方式传递的。当然，如果你传递了一个指向某个变量的指针，而函数对该指针执行了间接访问操作，那么函数就可以修改那个变量。尽管初看上去并不明显，但数组名作为参数时所发生的正是这种情况。这个参数（指针）实际上是通过传值方式传递的，函数得到的是该指针的一份拷贝，它可以被修改，但调用程序所传递的实参并不受影响。

### 声明数组参数
这里有一个有趣的问题。如果你想把一个数组名参数传递给函数，正确的函数形参应该是怎样的？它是应该声明为一个指针还是一个数组？

正如你所看到的那样，调用函数时实际传递的是一个指针，所以函数的形参实际上是个指针。但为了使程序员新手更容易上手一些，编译器也接受数组形式的函数形参。因此，下面两个函数原型是相等的：

```c
int　strlen( char *string );
int　strlen( char string[] );
```

这个相等性暗示指针和数组名实际上是相等的，但千万不要被它糊弄了！这两个声明确实相等，但只是在当前这个上下文环境中 。如果它们出现在别处，就可能完全不同，就像前面讨论的那样。但对于数组形参，你可以使用任何一种形式的声明。

你可以使用任何一种声明，但哪个“更加准确”呢？答案是指针。因为实参实际上是个指针，而不是数组。同样，表达式sizeof string的值是指向字符的指针的长度，而不是数组的长度。

## 总结
在绝大多数表达式中，数组名的值是指向数组第1个元素的指针。这个规则只有两个例外。sizeof返回整个数组所占用的字节而不是一个指针所占用的字节。单目操作符&返回一个指向数组的指针，而不是一个指向数组第1个元素的指针的指针。

除了优先级不同以外，下标表达式array[value]和间接访问表达式*(array+(value))是一样的。因此，下标不仅可以用于数组名，也可以用于指针表达式中。不过这样一来，编译器就很难检查下标的有效性。指针表达式可能比下标表达式效率更高，但下标表达式绝不可能比指针表达式效率更高。但是，以牺牲程序的可维护性为代价获得程序的运行时效率的提高可不是个好主意。

指针和数组并不相等。数组的属性和指针的属性大相径庭。当我们声明一个数组时，它同时也分配了一些内存空间，用于容纳数组元素。但是，当我们声明一个指针时，它只分配了用于容纳指针本身的空间。

当数组名作为函数参数传递时，实际传递给函数的是一个指向数组第1个元素的指针。函数所接收到的参数实际上是原参数的一份拷贝，所以函数可以对其进行操纵而不会影响实际的参数。但是，对指针参数执行间接访问操作允许函数修改原先的数组元素。数组形参既可以声明为数组，也可以声明为指针。这两种声明形式只有当它们作为函数的形参时 才是相等的。

数组也可以用初始值列表进行初始化，初始值列表就是由一对花括号包围的一组值。静态变量（包括数组）在程序载入到内存时得到初始值。自动变量（包括数组）每次当执行流进入它们声明所在的代码块时都要使用隐式的赋值语句重新进行初始化。如果初始值列表包含的值的个数少于数组元素的个数，数组最后几个元素就用缺省值进行初始化。如果一个被初始化的数组的长度在声明中未给出，编译器将使这个数组的长度设置为刚好能容纳初始值列表中所有值的长度。字符数组也可以用一种很像字符串常量的快速方法进行初始化。

多维数组实际上是一维数组的一种特型，就是它的每个元素本身也是一个数组。多维数组中的元素根据行主序进行存储，也就是最右边的下标率先变化。多维数组名的值是一个指向它第1个元素的指针，也就是一个指向数组的指针。对该指针进行运算将根据它所指向数组的长度对操作数进行调整。多维数组的下标引用也是指针表达式。当一个多维数组名作为参数传递给一个函数时，它所对应的函数形参的声明中必须显式指明第2维（和接下去所有维）的长度。由于多维数组实际上是复杂元素的一维数组，一个多维数组的初始化列表就包含了这些复杂元素的值。这些值的每一个都可能包含嵌套的初始值列表，由数组各维的长度决定。如果多维数组的初始化列表是完整的，它的内层花括号可以省略。在多维数组的初始值列表中，只有第1维的长度会被自动计算出来。

我们还可以创建指针数组。字符串的列表可以以矩阵的形式存储，也可以以指向字符串常量的指针数组形式存储。在矩阵中，每行必须与最长字符串的长度一样长，但它不需要任何指针。指针数组本身要占用空间，但每个指针所指向的字符串所占用的内存空间就是字符串本身的长度。

## 编程练习
### 练习 1
单位矩阵(identity matrix)就是一个正方形矩阵，它除了主对角线的元素值为1以后，其余元素的值均为0。编写一个名叫identity_matrix的函数，它接受一个10×10整型矩阵为参数，并返回一个布尔值，提示该矩阵是否为单位矩阵。

```c
#include <stdbool.h> // 引入stdbool.h以使用布尔类型(bool)
#include <stdio.h>

// 函数声明
bool identity_matrix(int matrix[10][10]);

// 主函数，示例用途
int main() {
    // 定义一个10x10的单位矩阵
    int matrix[10][10] = {
        {1, 0, 0, 0, 0, 0, 0, 0, 0, 0},
        {0, 1, 0, 0, 0, 0, 0, 0, 0, 0},
        {0, 0, 1, 0, 0, 0, 0, 0, 0, 0},
        {0, 0, 0, 1, 0, 0, 0, 0, 0, 0},
        {0, 0, 0, 0, 1, 0, 0, 0, 0, 0},
        {0, 0, 0, 0, 0, 1, 0, 0, 0, 0},
        {0, 0, 0, 0, 0, 0, 1, 0, 0, 0},
        {0, 0, 0, 0, 0, 0, 0, 1, 0, 0},
        {0, 0, 0, 0, 0, 0, 0, 0, 1, 0},
        {0, 0, 0, 0, 0, 0, 0, 0, 0, 1}
    };

    // 检查矩阵是否为单位矩阵
    if (identity_matrix(matrix)) {
        printf("The matrix is an identity matrix.\n");
    } else {
        printf("The matrix is not an identity matrix.\n");
    }

    return 0;
}

// identity_matrix函数定义
bool identity_matrix(int matrix[10][10]) {
    for (int i = 0; i < 10; i++) {
        for (int j = 0; j < 10; j++) {
            // 如果在主对角线上，检查元素是否为1
            if (i == j && matrix[i][j] != 1) {
                return false;
            }
            // 如果不在主对角线上，检查元素是否为0
            else if (i != j && matrix[i][j] != 0) {
                return false;
            }
        }
    }
    // 如果所有检查都通过了，则是单位矩阵
    return true;
}

```

### 练习 2
修改前一个问题中的identity_matrix函数，它可以对数组进行扩展，从而能够接受任意大小的矩阵参数。函数的第1个参数应该是一个整型指针，你需要第2个参数，用于指定矩阵的大小。

```c
#include <stdbool.h> // 引入stdbool.h以使用布尔类型(bool)
#include <stdio.h>  // 引入标准输入输出库以使用printf函数

// 函数声明
bool identity_matrix(int* matrix, int size);

// 主函数，示例用途
int main() {
    // 定义一个5x5的单位矩阵
    int matrix[5][5] = {
        {1, 0, 0, 0, 0},
        {0, 1, 0, 0, 0},
        {0, 0, 1, 0, 0},
        {0, 0, 0, 1, 0},
        {0, 0, 0, 0, 1}
    };

    // 检查矩阵是否为单位矩阵
    if(identity_matrix((int*)matrix, 5)) {
        printf("The matrix is an identity matrix.\n");
    } else {
        printf("The matrix is not an identity matrix.\n");
    }

    return 0;
}

// identity_matrix函数定义
bool identity_matrix(int* matrix, int size) {
    for (int i = 0; i < size; i++) {
        for (int j = 0; j < size; j++) {
            // 获取当前元素
            int elem = *(matrix + i*size + j);
            // 如果在主对角线上，检查元素是否为1
            if (i == j && elem != 1) {
                return false;
            }
            // 如果不在主对角线上，检查元素是否为0
            else if (i != j && elem != 0) {
                return false;
            }
        }
    }
    // 如果所有检查都通过了，则是单位矩阵
    return true;
}
```

### 练习 3
皇后是国际象棋中威力最大的棋子。皇后可以攻击位于箭头所覆盖位置的所有棋子。

我们能不能把8个皇后放在棋盘上，它们中的任何一个都无法攻击其余的皇后？这个问题被称为八皇后问题。你的任务是编写一个程序，找到八皇后问题的所有答案，看看一共有多少种答案。

```c
#include <stdio.h>
#include <stdlib.h>

#define N 8

// 用来记录每个答案的数量
int solutions = 0;

// 用来检查当前位置是否安全
int isSafe(int board[N][N], int row, int col) {
    int i, j;

    // 检查这一行的左边
    for (i = 0; i < col; i++)
        if (board[row][i])
            return 0;

    // 检查左上对角线
    for (i = row, j = col; i >= 0 && j >= 0; i--, j--)
        if (board[i][j])
            return 0;

    // 检查左下对角线
    for (i = row, j = col; j >= 0 && i < N; i++, j--)
        if (board[i][j])
            return 0;

    return 1;
}

// 递归函数来解决八皇后问题
int solveNQUtil(int board[N][N], int col) {
    // 如果所有皇后都被放置好了
    if (col >= N) {
        solutions++;
        printf("Solution %d:\n", solutions);
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < N; j++)
                printf(" %d ", board[i][j]);
            printf("\n");
        }
        printf("\n\n");
        return 1; 
    }
 
    int res = 0;
    for (int i = 0; i < N; i++) {
        // 检查放皇后在[i, col]是否安全
        if ( isSafe(board, i, col) ) {
            // 放置皇后
            board[i][col] = 1;
 
            // 递归到下一列
            res = solveNQUtil(board, col + 1) || res;
 
            // 如果放置皇后在[i,col]导致没有解决方案，就移除它
            board[i][col] = 0; // BACKTRACK
        }
    }
 
    // 如果这一列的所有行都没有找到解决方案，则返回0
    return res;
}

// 解决八皇后问题的函数
void solveNQ() {
    int board[N][N];
    // 初始化棋盘
    for(int i = 0; i < N; i++)
        for(int j = 0; j < N; j++)
            board[i][j] = 0;

    if (!solveNQUtil(board, 0)) {
        printf("Solution does not exist");
        return;
    }

    printf("Found %d solutions.\n", solutions);
}

// 主函数
int main() {
    solveNQ();
    return 0;
}

```















