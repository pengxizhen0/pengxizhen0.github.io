---
title: Maven依赖
categories:
  - 效率工具
  - Maven
tags:
  - Maven依赖
date: 2019-08-05 02:04:00
---

# Maven依赖

<!-- more --> 

## 依赖传递性

一般情况下，一个项目的依赖会从它的parent pom和dependency中得到。在允许依赖传递的情况下，整个项目的依赖关系会膨胀得非常快。以下几个因素为影响依赖的传递：

1. 依赖仲裁：一个artifact有多个版本的情况下，Maven会选择距离本项目最近的版本，如果两个依赖的距离是一样的，那么就选择第一个加载的依赖（这个加载顺序视情况而定）。A -> B(1.0)和A -> C -> B(2.0)，这两种情况下，项目A会使用依赖B1.0版本。
2. *Dependency management*： A -> B和A -> C -> B，如果我们在A中新增Dependency management，不管这两个B使用什么版本，都以Dependency management中的版本为准。
3. *Dependency scope*: 我们可以为依赖指定scope，每个scope的依赖传递性不同。
4. *Excluded dependencies*: 我们可以Exclude掉依赖的依赖。
5. *Optional dependencies*: 如果将一个pom A中的依赖B指定为optional，那么在其他项目中引用pom A，依赖B不会被自动引入，需要显式地引入B。



## Dependency management

Dependency management主要用于统一管理依赖的版本。在项目中要引用Dependency management里面的依赖，最小的坐标是**{groupId, artifactId, type, classifier}**，在默认情况下，type=jar，classifier=null。一般情况下使用**{groupId, artifactId}**即可。

由于一个pom只有一个parent，在小规模项目中，可以在parent pom中设置Dependency management。对于多模块项目，就不能引入多个parent pom了。可以利用scope的import引入多个Dependency management。这里的每个pom可以被称为"bill of materials" (BOM)，我们将一组相关的依赖打包成bom pom。下面例举了两个项目的bom以及他们的使用方法。对于import的scope，只是将该依赖用该依赖的内容进行替换。

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-framework-bom</artifactId>
            <version>4.0.1.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <dependency>
            <groupId>org.jboss.bom</groupId>
            <artifactId>jboss-javaee-6.0-with-tools</artifactId>
            <version>${some.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

