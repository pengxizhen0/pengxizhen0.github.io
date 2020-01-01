---
title: String_intern
categories:
  - Java知识
tags:
  - JDK源码
date: 2019-12-19 17:18:05
---

# String intern

String类中有个`intern()`的native，该方法会查询常量池中是否存在该字符串，若存在，则返回；若不存在则加入常量池并返回。在实现细节方面，JDK6又是与JDK7是不一样的。

<!-- more --> 

```java
public native String intern();
```



下面以一段常见的demo来说明问题。

String的创建：

1. 直接使用双引号声明出来的`String`对象会直接存储在常量池中。 

2. 使用new关键字创建的`String`对象存储在堆中。

```java
public static void main(String[] args) {
    //出现了字面量"1"，这个会被存储在常量池中
    //s指向的对象在堆中
    String s = new String("1");
    
    //因为常量池中已存在"1"，所以该语句无影响
    s.intern();
    
    //s2指向的对象在常量池中
    String s2 = "1";
    
    //false
    System.out.println(s == s2);
    
    //常量池中已经存在"1"
    //s3指向堆中的对象，值为"11"
    String s3 = new String("1") + new String("1");
    
    //JDK6和7的区别点就在这里
    //JDK6:"11"不存在常量中，创建"11"并放入常量池
    //JDK7:"11"不存在常量中，在常量池中放入s3
    s3.intern();
    
    //JDK6:s4指向的对象在常量池中
    //JDK7:s4指向的对象就是s3
    String s4 = "11";
    
    //JDK6:false
    //JDK7:true
    System.out.println(s3 == s4);
}
```

该程序在JDK6上的内存结构图

{% asset_img String_intern1.png intern1 %}



该程序在JDK7上的内存结构图

{% asset_img String_intern2.png intern2 %}

```java
public static void main(String[] args) {
    
    String s = new String("1");
    s.intern();
    String s2 = "1";
    System.out.println(s == s2);
    
    String s3 = new String("1") + new String("1");
    
    //以下两句交换顺序
    //"11"不存在常量池中，于是创建"11"对象放入常量池
    //这里要与intern区分，因为是用字面量创建s4，所以会创建"11"对象
    String s4 = "11";
    s3.intern();
    
    //JDK7:false
    System.out.println(s3 == s4);
}
```

该程序在JDK7上的内存结构图

{% asset_img String_intern3.png intern3 %}



在不同的JDK中，常量池存放的地方也不同

JDK6：常量池在永久代中

JDK7：常量池在堆中

JDK8：常量池在元空间中

其中**永久代**和**元空间**都是方法区的一种HotSpot VM实现，方法区是JVM所定义的规范。



结合常量池的存放位置，可以更好地理解intern()行为的区别。在JDK6中，永久代和堆不一样，所以常量池无相应字符串时会创建新对象。在JDK7中，由于都在堆中，常量池会引用堆中的字符串对象，不会创建新对象。



## Reference

https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html 