---
tags: ['C++']
title: C++速成：数据类型
created: '2022-01-28'
categories: 语言学习
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220129030603.png
---

# 基本类型
## 变量
&emsp;&emsp;变量存在的意义是对不同的内存块进行命名、分类和操作。每个变量就是一小块内存。我们知道不同的变量可以用不同的数据类型来声明，比如说浮点数（单/双精度）、整型数据等等。C++有以下常见的数据类型：
- bool
布尔类型，同Java的Boolean和C#的bool
- char
字符类型
- int
4byte长的整型数据类型
- short （int）
2byte长的短整型数据类型
- longlong  （int）
8byte长的长整型数据类型
- float
4byte长的单精度浮点数
- double
8byte长的双精度浮点数
- void
无类型
- wchar_t
宽字符型，实际上的内存空间与short int等价，占2或4byte

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220209204908.png" width=700/>

&emsp;&emsp;我们可以**用typeof来给数据类型命名**。例如：
```C++
typedef int feet;
```
&emsp;&emsp;之后我们就可以用feet a = 10;来代替int a = 10;了。
&emsp;&emsp;说了这么多我们好像没有提到字符串（Java的String，C#的string），在IO流中输出字符串的时候也是直接用双引号就输出了。在C++中声明一串字符的方法有两种：
- C风格的字符数组，**数组符号是标记在变量名上的**
```C++
char myStr1[] = "Hello";
```
- C++风格的字符串，**需要引用string头文件**
```C++
#include <string>;
...
string myStr2 = "World";
```

## 常量
&emsp;&emsp;对于不可变的数据，我们可以用宏（#define）或者constant定义：
```C++
#define a = 10;
#include <iostream>;
using namespace std;

int main(){
    constant int b = 20;
    return 0;
}
```
&emsp;&emsp;当修改常量时会报错。

# 用户输入变量
&emsp;&emsp;我们可以利用IO流的I（cin）来获取用户输入的数据：
```
    int a = 0;
    cout << "请输入a的值: " << endl;
    cin >> a;
    cout << "a = " << a << endl;
```

# 数组
这部分的定义规则和Java很类似，但是有个需要注意的点就是C++或者C的**数组的中括号是放在变量名后面的，而不是类型关键字后面**。
```C++
int arr1[] = {1,2,3,4,5,6};
int arr2[][] = {
  {1,2,3},
  {4,5,6}};
```
除此之外，我们获取数组长度需要另辟蹊径，因为没有length方法。我们可以借用sizeof(arr)/sizeof(arr[0])来获取数组长度。sizeof的含义是占用字节的多少。

# Enum类型
&emsp;&emsp;枚举类型，和其他的高级程序语言一样表示一组用户定义类型的枚举集合：
```C++
enum color { red, green, blue } c;
c = red;
```
&emsp;&emsp;这代表枚举集合“color”中包含了red、green、blue三种类型，变量c是color集合的一个变量，值为red。类型具体是什么值并不重要，我们通常可以用枚举来进行状态机的模拟，只需要一个状态就好了。


