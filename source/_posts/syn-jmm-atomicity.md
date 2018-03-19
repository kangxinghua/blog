---
title: 同步和Java内存模型 (二)原子性
date: 2016-11-15 14:58:00
tags: 笔记
---
除了long型字段和double型字段外，java内存模型确保访问任意类型字段所对应的内存单元都是原子的。这包括引用其它对象的引用类型的字段。此外，volatile long 和volatile double也具有原子性 。（虽然java内存模型不保证non-volatile long 和 non-volatile double的原子性，当然它们在某些场合也具有原子性。）（译注：non-volatile long在64位JVM，OS，CPU下具有原子性）
<!-- more -->
当在一个表达式中使用一个non-long或者non-double型字段时，原子性可以确保你将获得这个字段的初始值或者某个线程对这个字段写入之后的值；但不会是两个或更多线程在同一时间对这个字段写入之后产生混乱的结果值（即原子性可以确保，获取到的结果值所对应的所有bit位，全部都是由单个线程写入的）。但是，如下面（译注：指可见性章节）将要看到的，原子性不能确保你获得的是任意线程写入之后的最新值。 因此，原子性保证通常对并发程序设计的影响很小。  

> 原文：http://gee.cs.oswego.edu/dl/cpj/jmm.html 第二章  
> 作者：Doug Lea 译者：程晓明  校对：方腾飞  
