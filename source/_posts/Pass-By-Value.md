---
title: Java中的参数传递——值传递、引用传递
date: 2017-02-15 09:59:42
tags: 笔记
---
首先，不要纠结于Pass By Value 和 Pass By Reference 的字面上的意义，否则很容易陷入所谓的“一切传引用其本质上是传值”这种并不能解决问题毫无意义论战中。更何况，要想知道Java到底是传值还是传引用，起码你要先知道传值和传引用的准确含义吧？可是如果你已经知道了这两个名字的准确含义，那么你自己就能判断Java到底是传值还传引用。这个好像用大学的名词来解释高中的题目，对于初学者根本没有任何意义。  
<!-- more -->
- 搞清楚基本类型和引用类型的不同之处
``` java 
int num = 10;
String str = "hello";
``` 
![alt](166032bc90958c21604110441ad03f45_r.jpg)  
如图所示，num是基本类型，值就直接保存在变量中。而str是引用类型，变量中保持的只是实际对象的地址。一般称这种变量为“引用”，引用指向实际对象，实际对象中保存着内容。

- 搞清楚赋值运算符（=）的作用  
``` java 
num = 20;
str = "java";
```
![alt](287c0efbb179638cf4cf27cbfdf3e746_b.jpg) 

对于基础类型num，赋值运算符会直接改变变量的值，原来的值被覆盖掉。
对于引用类型str，赋值运算符会改变引用中所保存的地址，原来的地址被覆盖掉。**但是原来的对象不会被改变（重要）。**  
如上图所示，“hello” 字符串对象没有被改变。（没有被任何引用所指向的对象是垃圾，会被垃圾回收器回收）  
- 调用方法时候发生了什么？**参数传递基本上就是赋值操作。**
``` java 
//第一个例子：基础类型 
void foo(int value){
    value = 100；
}
foo(num);//num 没有被改变 

//第二个例子：没有提供改变自身方法的引用类型 
void foo(String text){
    text ="windows";
}
foo(str);//str 也没被改变 

//第三个例子：提供了改变自身方法的引用类型
StringBuilder sb = new StringBuilder("iphone");
void foo(StringBuilder builder){
    builder.append("4");
} 
foo(sb);//sb 被改变了，变成了“iphone4”。

//第四个例子：提供了改变自身方法的引用类型，但是不是使用，而是使用赋值运算符。
StringBuilder sb = new StringBuilder("iphone");
void foo(StringBuilder builder){
    builder = new StringBuilder("ipad");
}
foo(sb);//sb 没有被改变，还是“iphone”。
```

重点理解为什么，第三个例子和第四例子结果不同？
下面是第三个例子的图解：
![alt](d8b82e07ea21375ca6b300f9162aa95f_b.jpg)

builder.append("4")之后

![alt](ff2ede9c6c55568d42425561f25a0fd7_b.jpg)

下面是第四个例子的图解：

![alt](d8b82e07ea21375ca6b300f9162aa95f_b.jpg)

builder = new StringBuilder("ipad");之后

![alt](46fa5f10cc135a3ca087dae35a5211bd_b.jpg) 


2017-02-15 11:09:20 康兴华
发现以前一直以来的理解都是错误的，单纯说方法传递是传值和和传引用毫无意义的。



