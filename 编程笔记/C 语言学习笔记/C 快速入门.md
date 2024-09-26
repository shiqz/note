## 前言
C 语言是一种通用的、面向过程式的计算机程序设计语言。1972 年，为了移植与开发 UNIX 操作系统，丹尼斯·里奇在贝尔电话实验室设计开发了 C 语言。

C 语言的应用非常广泛，例如：操作系统、语言编译器、汇编器、文本编辑器、打印机、网络驱动器、现代程序、数据库、语言解释器、实体工具。

一个 C 语言程序，可以是 3 行，也可以是数百万行，它可以写在一个或多个扩展名为 ".c" 的文本文件中，例如，hello.c。您可以使用 "vi"、"vim" 或
任何其他文本编辑器来编写您的 C 语言程序。

## 第一个C程序
创建一个文件名为：`hello.c` 的文件，内容如下：
```c
#include <stdio.h>

int main()
{
    /* 我的第一个 C 程序 */
    printf("Hello, World! \n");
    

    return 0;
}
```
其中 `#include` 是预处理命令，用来引入头文件，而这里的 `<stdio.h>` 是标准输入输出的头文件，它提供了：`printf` 函数。
> 如果我们要使用输入输出，则必须引入 `stdio.h` 头文件，否则会发生编译错误。

注意：在所有C程序中，有且仅有一个 `main` 函数，它作为整个程序的入口。这里的 `int` 表示是 `main` 函数的返回值，通常就是一个整数
返回 0 表示没有任何错误。

`/* ... */` 是块注释。

`//` 是单行注释。

### 如何编译和执行
首先先安装 `gcc` 编译器：
```bash
gcc hello.c
# 此时会生成 a.out（windows 会生成 a.exe）

./a.out
# Hello, World!

# 编译到到指定文件
gcc hello.c -o hello.exe

./hello.exe
# Hello, World!
```

## Token
- **关键字**（Keywords）
- **标识符**（Identifiers）
- **常量**（Constants）
- **字符串字面量**（String Literals）
- **运算符**（Operators）
- **分隔符**（Separators）
