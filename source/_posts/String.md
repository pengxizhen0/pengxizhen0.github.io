---
title: String
categories:
  - Java知识
tags:
  - JDK源码
date: 2019-12-20 13:03:54
---

# String、StringBuilder、StringBuffer

String类与其中的成员变量value是final的，value是一个定长的char数组，代表着String一旦初始化好就不能改变。一切对原String的修改操作都会创建新的String对象。

<!-- more --> 

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



StringBuilder和StringBuffer都继承了抽象类AbstractStringBuilder，在抽象类中，value是一个变长的数组，增加字符串时，若char数组空间足够，则直接添加；若空间不够，则会new一个更大的数组，将旧数据搬移过去。由于搬移数据是native方法，使用指针寻址来保证效率。

StringBuilder和StringBuffer的区别就在于，前者是线程不安全的，后者是线程安全的，线程安全的方法就是在整个方法上加`synchronized`。

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    char[] value;
    int count;
    //...
}
```

