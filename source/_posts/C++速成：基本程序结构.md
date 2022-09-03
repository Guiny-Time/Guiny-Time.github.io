---
tags: [语言学习/C++]
title: C++速成：基本程序结构
created: '2022-02-10'
categories: 语言学习
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220129030603.png
---

# 条件判断语句
&emsp;&emsp;和其他语言类似，不细说了：
```C++
// 单行if
if(a > 10){
  cout << "你是嘟嘟猪" << endl;
}

// if-else
if(b > 10){
  cout << "你是嘟嘟猪" << endl;
}else{
  cout << "我是奶油" << endl;
}

// if-else if-else
if(c > 10){
  cout << "你是嘟嘟猪" << endl;
}else if(c >= 5 && c<=10){
  cout << "我是奶油" << endl;
}else{
  cout << "好耶" << endl;
}

// if套娃
if(d >= 0){
  cout << "Your first step." << endl;
  if(d >= 100){
    cout << "Great." << endl;
  }
}

// 三元运算符
e >= 10 ? cout << "Good. the variable is bigger than 10." << endl : cout << "Oh no." << endl;
```

# 选择语句
&emsp;&emsp;尽量少用。在《代码整洁之道》里对switch的评价是：**switch惊悚现身**。
&emsp;&emsp;还是和其他语言类似（所以要注意的点就是break的问题），不细说了：
```C++
switch(a){
  case 1:
    cout << "I am One." << endl;
    break;
  case 2:
    cout << "I am Two." << endl;
    break;
  case 3:
    cout << "I am Three." << endl;
    break;
  default:
    cout << "I am nothing." << endl;
    break;
}
```

# 循环语句
&emsp;&emsp;还是差不多，不多说了：
```C++
// while条件循环
while(a<5){
  a++;
  cout << "a plus plus: " + a << endl;
}

// do-while条件循环
do{
  a++;
  cout << "a plus plus: " + a << endl;
}while(a<5);

// for循环
for(int i = 0; i < 5; i++){
  a++;
  cout << "a plus plus: " + a << endl;
}
```

#  流程结构关键字：break、continue和goto
&emsp;&emsp;前两个关键字对大家来说都是比较熟悉的（和其他语言一致）：
- break指终止并跳出当前操作（例如跳出switch、跳出循环）
- continue指跳过代码块位于该关键词之后的语句并继续执行（例如跳过部分循环语句，然后继续循环直到循环结束）
这样可以用来在循环语句中进行快速筛选（遇到某些条件直接continue）

&emsp;&emsp;而goto是我第一次遇到，是一种**比较危险**的语句，他用来执行跳转的功能（人如其名）。
```C++
goto AAA;
...
...
...
AAA:
...
...
...
```
&emsp;&emsp;使用不当的时候会出现死循环（比如说跳转到了goto前面，又执行了一遍goto，又跳转又执行...）

