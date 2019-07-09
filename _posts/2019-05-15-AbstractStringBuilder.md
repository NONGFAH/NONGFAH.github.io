---
layout:     post
title:      java.lang.AbstractStringBuilder
subtitle:   本文为 Java 源码阅读中的一节，重要程度为 1 
date:       2019-05-15
author:     NONGFAH
header-img: img/post-bg-mma-0.png
catalog: true
tags:
    - JDK
    - 源码
---
> 根据JDK1.8中的源码注释，AbstractStringBuilder 首次出现在1.5版本，但由于其类访问权限为 default --包访问权限，只能在包内访问，外包代码无法使用。
故目前官方文档中没有同步这个抽象类，并将其子类 StringBuilder 、StringBUffer 的父类定义为 Object。

# 类定义  
AbstractStringBuilder 类被设计为 StringBuffer、 StringBuilder 类的抽象基类，并且访问权限为 default --包访问权限。  
#### 接口  
该类实现了三个接口：
- Appendable ：包含 append(CharSequence csq) 添加字符序列; append(CharSequence csq, int start, int end) 添加字符序列的一部分;append(char c)添加字符;  
- CharSequence : 包含 length() , charAt(int) , subSequence(int,int),toString() ;表示char值的一个可读序列

#### 方法  
AbstractStringBuilder 类包括用于操作序列的方法：
- append() 用于在其后追加内容
- insert() 用于在其中插入内容
- delete() 用于删除指定位置或指定区间的内容
- trimToSize() 用于容量缩减到实际所用大小
- ensureCapacity() 用于扩容
- substring() 用于提取子串
- length() 返回实际长度
- capacity() 返回现有容量

#### 存储结构  
该类的存储结构是一个可变的 char 数组
# 属性  

##### char[] value;
用于存储 实际内容，包访问权限，可变。  
##### int count;
数组实际使用的字符个数，区别于数组的容量。


# 方法

由于 AbstractStringBuilder 底层数据结构与 String 相同， 其大部分方法也基本类似，都是使用 System.ArrayCopy() 对内部维护的
char[] 进行复制，各方法不再分析。简单介绍扩容算法

##### ensureCapacity(int min)
该方法比较 传入参数 与 容量 ，当传入参数大于0且大于内部容量时，使用内部私有方法将继续判断 传入参数 与（容量+1）*2 的大小
取其中较大值并判断是否溢出，如果出现溢出情况抛异常，否则将容量扩大到该较大值。

