---
title: Spring
categories:
  - Spring
tags:
  - 别名依赖
date: 2019-06-28 16:28:16
---

# Spring中如何避免bean别名循环依赖

在Spring bean的配置中，可以为bean配置一个别名。但在配置别名时，开发人员可能会不小心将别名配置成一个循环。Spring会检测出这种情况并且抛出一个异常。

<!-- more --> 

```xml
<!-- 这种循环的别名配置会抛异常 -->
<alias name="demoService" alias="alias1"/>
<alias name="alias1" alias="alias2"/>
<alias name="alias2" alias="demoService"/>
```



## 设置别名

在Spring中，与此相关的有两个Map。一个用于保存beanName与BeanDefinition的映射信息，一个用于保存aliasName与beanName的映射信息。

```java
/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

/** Map from alias to canonical name. */
private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
```

例如，当我们需要为A新增一个别名B时，为避免成环，会检测A是否是B的别名。若是的话，则会抛出异常。

```java
/**
* Determine whether the given name has the given alias registered.
* @param name the name to check
* @param alias the alias to look for
* @since 4.2.1
*/
public boolean hasAlias(String name, String alias) {
    for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
        String registeredName = entry.getValue();
        if (registeredName.equals(name)) {
            String registeredAlias = entry.getKey();
            if (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)) {
                return true;
            }
        }
    }
    return false;
}
```



## 使用别名

bean的别名可能会组成一列，即A的别名是B，B的别名是C。Spring用以下方式去找到别名的正式名。

```java
/**
* Determine the raw name, resolving aliases to canonical names.
* @param name the user-specified name
* @return the transformed name
*/
public String canonicalName(String name) {
    String canonicalName = name;
    // Handle aliasing...
    String resolvedName;
    do {
        resolvedName = this.aliasMap.get(canonicalName);
        if (resolvedName != null) {
            canonicalName = resolvedName;
        }
    }
    while (resolvedName != null);
    return canonicalName;
}
```

