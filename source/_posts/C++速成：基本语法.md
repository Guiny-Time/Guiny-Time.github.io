---
tags: ['C++']
title: C++速成：基本语法
created: '2022-01-28'
categories: 语言学习
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220129030603.png
---

&emsp;&emsp;怎么突然开始学C++ 了？主要是因为实习的岗位是特效图像图形算法，考虑到在手机上使用的图形API可能是OpenCV或者OpenGL ES，二者都需要C++ 作为前置知识。加上国内高校计算机学的都是C/C++ 路线，咱这边学的是Java...再看看Learn OpenGL书上的话：
> 由于OpenGL是一个图形API，并不是一个独立的平台，它需要一个编程语言来工作，在这里我们使用的是C++ 。所以，对C++的熟练掌握在学习这个教程中是必不可少的。当然，我仍将尝试解释大部分用到的概念，包括一些高级的C++话题，所以，你并不一定要是一个C++ 专家才能来学习。不过，请确保你至少应该能写个比 ‘Hello World’ 复杂的程序。

&emsp;&emsp;我决定速成一下cpp，主要在基本的语法和基本的数据结构上。

# 创建第一个cpp项目
&emsp;&emsp;我选的IDE是CLion，主要还是对JetBrains家感觉不错，功能应该是没话说的，在校期间又可以免费用。然后我们就可以选择C++14来创建我们的第一个项目，其中已经包含了一个控制台输出HelloWorld的语句：
```C++
#include <iostream>

int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}
```

## 头文件
&emsp;&emsp;第一行的include代表该文件使用的头文件，这里表示引入了名为iostream(io流)的头文件。include和C#的using或者Java的import类似。通常，我们还会引入C Standard Library、String：
```C++
#include <cstdlib>
#include <string>
```

&emsp;&emsp;cstdlib定义了数据流的输入和输出（cin、cout）；string允许我们使用字符串类。除此之外，还有其他几种常用的包：
- #include \<limits\>
允许返回函数的最大值，例如double的最大值、int的最大值等
- #include \<vector\>
允许使用vector数据结构。这是C++中非常强大的一种数据结构
- #include \<sstream\>
即string stream，字符串类型的流
- #include \<numeric\>
STL常用的数字操作，如accumulate等
- #include \<ctime\>
处理与时间有关的工作。或许它能够在OpenGL里大放异彩
- #include \<math\>
包含了大量数学运算的基本类

## 第一个输出
&emsp;&emsp;现在来看第一个输出的实现：
```C++
int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}
```
&emsp;&emsp;std表示的是文件使用的命名空间(namespace)，我们用命名空间来区分不同的模块，这和C#的namespace以及Java的package是一样的。调用带有命名空间的函数或变量需要使用“命名空间名::”的形式。
&emsp;&emsp;实际上，当文件中声明了 **“using namespace std;”** 语句之后，我们可以把上述代码简写为：
```C++
using namespace std;
int main() {
    cout << "Hello, World!" << endl;
    return 0;
}
```
&emsp;&emsp;main函数作为主程序执行的入口，是程序开始执行的地方，里面发生的是我们希望程序运行的内容（和Java的main函数类似）。int前缀代表main返回一个int类型的数据。通常，我们用这个数据（0）来判断程序是否正确运行。
&emsp;&emsp;**“<<”** 指的是io流的流向。在这个语句中，意思是“Hello, World!”语句流向输出函数**cout**。同样的，std命名空间下的分行函数**endl**也通过这个符号流向字符串，最终流向输出函数**cout**。
&emsp;&emsp;和其他大部分语言一样，C++的语句结尾也需要加上分号。

## 全局变量与局部变量
&emsp;&emsp;我们可以稍微改写一下我们的程序：
```C++
#include <iostream>
using namespace std;

int value = 1;

int main() {
    int x = 10;
    x = x + value;
    value++;
    x = x + value;
    cout << x << endl;    // 13

    return 0;
}
```
&emsp;&emsp;处于main函数之外的变量value即全局变量，所有处于std命名空间下的函数都可以使用和更改value的值。如果我们不希望其他函数更改全局变量的值，可以在前面加上constant关键字：
```C++
#include <iostream>
using namespace std;

constant int value = 1;

int main() {
    int x = 10;
    x = x + value;
    value++;
    x = x + value;
    cout << x << endl;    // 12

    return 0;
}
```
&emsp;&emsp;处于main函数内的变量x就是main函数下的局部变量，我们无法在外部使用x。