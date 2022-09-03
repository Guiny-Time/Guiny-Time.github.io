---
tags: ['C++']
title: C++速成：结构体
created: '2022-02-10'
categories: 语言学习
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220129030603.png
---

# 语法
&emsp;&emsp;其实在写Cg（C for graphic）代码的时候已经接触过结构体了：
```Cg
struct appdata_t {
    float4 vertex : POSITION;
    float2 texcoord: TEXCOORD0;
};
```
&emsp;&emsp;但C++的写法和Cg稍有不同，比如说我们要**在main函数上方**声明一个人的结构体并实例化一个结构体对象s3（一般不这么写）：
```C++
struct person {
    int age;
    string name;
}p3;
```
&emsp;&emsp;然后实例化对象出来（**struct关键字是可以省略掉的**）：
```C++
struct person p1 = {10, "ZhangSan"};
struct person p2;
p2.age = 20;
p2.name = "LiSi";
```

# 结构体数组
&emsp;&emsp;很自然的，我们可以创建结构体的数组并执行某些操作：
```C++
struct Person
{
	int age;
	string name;
};

int main() {
	Person p1 = { 10, "ZhangSan" };
	Person p2 = { 20, "LiSi" };
	Person p3 = { 30, "WangWu" };
	Person structArr[] = {p1, p2, p3};
	Person structArr2[3];

	for (int i = 0; i < 3; i++) {
		structArr2[i].age = 10 + i;
		structArr2[i].name = "Mr.Unknow";
		cout << "Hello, my name is " << structArr[i].name << ". And I am " << structArr[i].age << " years old." << endl;
		cout << "Hello! my name is " << structArr2[i].name << "! And I am " << structArr2[i].age << " years old!" << endl;
		cout << "" << endl;
	}
	return 0;
}
```
**输出结果**：
<img src="https://i.imgur.com/ujXVLKc.png"/>

# 结构体指针
&emsp;&emsp;我们可以将实例化出来的结构体实例与指针相绑定：
```C++
Person *p = &p1;
cout << p->age << endl;
```
&emsp;&emsp;当指针p与结构体p1绑定之后，我们就可以用**特殊的“->”操作**访问结构体内的变量

# 结构体嵌套
&emsp;&emsp;当然可以这么做，这属于一种包含的关系：
```C++
struct Student{
  int id;
  string name;
};

struct Teacher{
  int id;
  string name;
  Student std;
};
```
&emsp;&emsp;然后，我们可以这么干来获取子结构体的信息：
```C++
Student m = { 1000, "ZhangSan" };
Teacher teacherA = {10001, "XXX", m};
cout << "My student name is: " << teacherA.std.name << endl;
```
&emsp;&emsp;实际上这种一对多的关系，写在教师结构体内的应该是学生结构体数组才对。

> **写在文末的注意事项**
&emsp;&emsp;结构体作为函数参数的部分我没有提到，因为通过前文都可以很容易的总结出来，就不浪费笔记的空间了。但是有个需要注意的地方是：当我们希望节约内存而输入结构体指针，又希望输入函数的结构体是只读的时候，可以在函数输入参数前加上const关键字进行修饰：
void example(constant Student * std){}
&emsp;&emsp;这是在上一篇文章中提到过的常量指针，这代表指针指向的内存储存的值是不可改的（但是值相同的情况下，内存地址可以不同。具体请查看前文）。

