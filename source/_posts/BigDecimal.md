---
title: BigDecimal
categories:
  - Java知识
tags:
  - JDK源码
date: 2019-11-06 16:12:16
---

# BigDecimal和BigInteger

在商业上进行计算的时候，如果使用float、double型变量，会造成丢失精度的问题。这种情况下应该使用BigDecimal。

<!-- more --> 

## double

以双精度为例，单精度原理相同。

64bit的双精度浮点数分为以下三个部分：

- sign bit(符号): 用来表示正负号。

- exponent(指数): 用来表示次方数。

- mantissa(尾数): 用来表示精确度。

一个双精度浮点数的数值为：sign × 2^(exponent - 0x3ff) × 1.mantissa

mantissa从左到右第N位表示2^-N，从大到小依次为0.5、0.25、0.125....

浮点数的小数部分由2^-N组成，2^-N精度有限，并不能表示所有的小数。



## BigDecimal

BigDecimal采用小数变整数的方法解决精度问题。既然小数不能精确表示，那么就将小数点去掉，变为整数。整数可以用二进制精确表示。整数由intCompact体现，被丢掉的小数点由scale体现。



 {% asset_img BigDecimal.png BigDecimal %} 



## BigInteger

如果我们要使用的数超过了long的值范围，可以使用BigInteger类。BigInteger里有int数组，它所表示的数就是由数组中的int连接起来。