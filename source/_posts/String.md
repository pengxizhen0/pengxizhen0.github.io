---
title: Java中的字符串
categories:
  - Java知识
tags:
  - JDK源码
date: 2019-12-20 13:03:54
---

# 引言

在Java中与字符串有关的类有String、StringBuilder、StringBuffer。

<!-- more --> 



# String

String类与其中的成员变量value都是final的，value是一个定长数组，代表着String一旦初始化好就不能改变。一切对原String的修改操作都会创建新的String对象。

```java
public final class String 
    implements java.io.Serializable, Comparable<String>, CharSequence {
    
    private final char value[];
    private int hash;
    
    //以concat和substring为例子说明修改操作都会返回一个新的String对象
    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
    
    public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
}
```



## 为什么String是不可变对象

1. 性能。String会被保存到常量池中，相同的String会指向常量池中相同的对象。在保证性能的背景下，String对象需要是不可变的。可变的String会造成线程不安全，例如在HashMap的使用场景中，用String作Key，当该Key被某一线程修改时，其他线程会出现找不到Key和Value的情况。
2. 安全。String在Java中的用处广泛，例如操作某个文件、打开某个连接、加载某个类等，这些操作都会使用String。若String可变的话，在进行这些操作时，文件名、连接名、类名可能会被有意或无意地改动。
3. 省内存。不可变的String保存在常量池中可以节省内存使用。



# StringBuilder与StringBuffer

StringBuilder和StringBuffer都继承了抽象类AbstractStringBuilder，在抽象类中，value是一个变长数组，增加字符串时，若char数组空间足够，则直接添加；若空间不够，则会new一个更大的数组，将旧数据搬移过去。由于搬移数据是native方法，使用指针寻址来保证效率。

StringBuilder和StringBuffer的区别就在于，前者是线程不安全的，后者是线程安全的，线程安全的方法就是在整个方法上加`synchronized`。因此在使用性能上，StringBuilder是好于StringBuffer的。

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    char[] value;
    int count;
    //...
}
```



# 总结

String、StringBuilder、StringBuffer都是使用`char`数组来保存字符串，区别是String的`char[]`是不可变对象，其余两者是可变对象。

另外StringBuilder和StringBuffer的区别是，线程的安全与否，线程安全的类中所有方法都使用`synchronized`修饰。