# 元注解
元注解的作用就是负责注解其他注解。Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对自定义annotation类型作说明。Java5.0定义的元注解：
1. @Target
2. @Retention
3. @Documented
4. @Inherited

这些类型和他们所支持的类在java.lang.annotation包中可以找到。下面我们看一下每个元注解的作用和相应参数的使用说明。

## @Target
@Target说明了annotation所修饰的对象的范围：annotation可被用于packages、types（类、接口、枚举、Annotation类型）、类型成员（方法、构造方法、成员变量、枚举值）、方法参数和本地变量（如循环标量、catch参数）。在Annotation类型的声明中使用了target可以更加明晰其修饰的目标。

作用：用于描述注解的使用范围（即：被描述的注解可以用在什么地方）

取值（ElementType）有：
1. CONSTRUCTOR：用于描述构造器
2. FIELD：用于描述成员变量（包括枚举常量）
3. LOCAL_VARIABLE：用于描述局部变量
4. METHOD：用于描述方法
5. PACKAGE：用于描述包
6. PARAMETER：用于描述参数
7. TYPE：用于描述类、接口（包括注解类型）或enum声明

## @Retention
@Retention定义了该annotation被保留的时间长短：某些annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的annotation可能会被虚拟机忽略，而另一些在class被装载时将被读取（annotation并不影响class的执行，因为annotation与class在使用上是被分离的）。使用这个meta-annotation可以对annotation的“生命周期”进行限制。

作用：表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注解在什么范围内有效）

取值（RetentionPolicy）有：
1. SOURCE：在源文件中有效（即源文件保留）
2. CLASS：在class文件中有效（即class文件保留，默认值）
3. RUNTIME：在运行时有效（即运行时保留）

Retention meta-annotation类型有唯一的value作为成员，它的取值来着java.lang.annotation.RetentionPolicy的枚举类型值。具体事例如下：
```java
package com.mec.hl7.dao;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    String name() default "";
}
```
Column注解的RetentionPolicy的属性值是RUNTIME，这样可以通过反射获取到该注解的属性值，从而去做一些运行时的逻辑处理

## @Documented
@Documented注解表明这个注解应该被javadoc工具记录。默认情况下javadoc中不会包含注解。但如果声明注解时指定了@Documented，则他会被javadoc之类的工具处理，注解类型的信息也会被包含到生成的文档中。@Documented是一个标记注解，没有成员。

## @Inherited
@Inherited元注解是一个标记注解，它阐述了某个被标记的注解是可以被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

注意：被@Inherited标记的注解如果被用于注解类以外的任何内容，则@Inherited将会失效。另外被@Inherited标记的注解只会被标注过的class的子类所继承，类并不从它所实现的接口继承annotation。

当@Inherited annotation类型标注的annotation的Retention是RetentionPolicy.RUNTIME，则反射API增强了这种继承性。如果我们使用java.lang.reflect去查询一个@Inherited annotation类型的annotation时，反射代码检查将展开工作：检查class和其父类，直到发现指定的annotation类型被发现，或者到达类继承结构的顶层。

# 自定义注解
使用@interface自定义注解时，自动继承了java.lang.annotation.Annotation接口，由编译程序自动完成其他细节。在定义注解时，不能继承其他的注解或接口。@interface用来声明一个注解，其中的每个方法实际上是声明一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型（返回值类型只能是基本类型、Class、String、enum）。可以通过default来声明参数的默认值。

定义注解格式：`public @interface 注解名 {定义体}`

注解参数的可支持数据类型：
1. 所有基本数据类型（int, float, boolean, byte, double, char, long, short）
2. String类型
3. Class类型
4. enum类型
5. Annotation类型
6. 以上所有类型的数组

Annotation类型里面的参数设定：
1. 只能用public或默认（default）这两个访问权修饰。例如，String value();这里把方法设为default默认类型；
2. 参数成员类型只能是以上列出的6种类型。例如，String value();这里的参数成员类型为String；
3. 如果只有一个参数成员，最好把参数名称设为value。例如，String value();

> http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html