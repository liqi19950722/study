# Java模块化

动机：

- 强封装

- 安全提升

- 增快应用模块中类型检测的速度

- 瘦身JDK

基础

定义模块

```java
module java.compiler {
    exports javax.annotation.processing;
    exports javax.lang.model;
    exports javax.lang.model.element;
    exports javax.lang.model.type;
    exports javax.lang.model.util;
    exports javax.tools;

    uses javax.tools.DocumentationTool;
    uses javax.tools.JavaCompiler;
}
```

模块依赖

```java
module java.sql {
    requires transitive java.logging;//transitive 传递依赖
    requires transitive java.transaction.xa;
    requires transitive java.xml;

    exports java.sql;//exports 控制可访问的API package
    exports javax.sql;

    uses java.sql.Driver;
}
```



模块路径

类路径

- 通过artifacts的ClassPath区分类型
- 无法区分artifacts
- 无法提前通知artifacts缺少
- 允许不同的artifacts定义在相同的packages定义类型

- 差异性
  - 定义整个模块而非类型
  - 无论是运行时，还是编译时，在同一目录下不允许出现同名模块

模块解析

可读性

可访问性



迁移

- 明确应用实现的依赖JDK模块
- 明确二方或三方JAR所依赖的JDK模块
- 需要微服务化应用

模块

- 命名模块

  ```java
  requires spring.core;
  requires spring.context;
  requires org.apache.commons.lang3;
  ```

- 非命名模块

  ```java
  requires java.base;//默认依赖
  requires java.compiler;//exports 控制可访问的API package
  requires java.sql;
  ```

- 自动模块`MANIFEST.MF`

  ```java
  Manifest-Version: 1.0
  Implementation-Title: spring-context
  Automatic-Module-Name: spring.context
  Implementation-Version: 5.1.4.RELEASE
  Created-By: 1.8.0_144 (Oracle Corporation)
  ```

  

模块化反射

`java.lang.Module`

`java.lang.module.ModuleDescriptor`

```java
 public static enum Modifier {
     OPEN,
     AUTOMATIC,
     SYNTHETIC,
     MANDATED
 }
public final static class Exports implements Comparable<Exports>{...}
public final static class Opens implements Comparable<Opens>{...}
public final static class Provides implements Comparable<Provides>{...}
public final static class Version implements Comparable<Version>{...}
public final static class Requires implements Comparable<Requires>{...}
public final static class Version implements Comparable<Version>{...}
```



## 附

Annotation-Processing Tool(APT)

https://docs.oracle.com/javase/7/docs/technotes/guides/apt/

http://openjdk.java.net/jeps/117

Java SE Embedded 8 Compact Profiles

https://blogs.oracle.com/jtc/an-introduction-to-java-8-compact-profiles

https://www.oracle.com/technetwork/java/embedded/resources/tech/compact-profiles-overview-2157132.html

