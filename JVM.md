# Class File Format

# 类加载

## Loading
当我们使用一个ClassLoader加载一个类时，先会在当前类加载器中寻找是否已经加载过此类，有则直接返回。如果没有，则会委托父加载器去加载。父加载器同样先在自己的加载器中寻找是否已加载过此类，有则直接返回，如果也没有则继续委托给自己的父加载器去加载。直到最顶层的类加载器BoostrapClassLoader，如果仍然没有找到指定的类，BootstrapClassLoad先会尝试加载，如果无法加载，则会委托给子加载器去加载，如果仍然不行则继续向下委托，即，双亲委派。

### 自定义类加载器
自定义ClassLoader，只需要重写findClass方法。（class文件二进制数据被加载进内存，然后通过defineClass方法定义一个Class类型的对象）
重写loadClass方法可以打破双亲委派机制。

## Linking
- Verification [ˌvɛrəfəˈkeɪʃən] ：验证文件是否符合JVM规范。
- Preparation [ˌprepəˈreɪʃn] ：静态成员变量（不包括实例变量）赋默认值。默认值的意思不同数据类型的零值（0，0.0，null，false）。特殊情况，如果变量被final关键字修饰，那么在该阶段变量将会被赋初始值。
- Resolution：将类、方法、属性等符号引用替换为直接引用。虚拟机将常量池内的符号引用替换为直接引用的过程，也就是得到类或者字段、方法在内存中的指针或者偏移量。

## Initializing
调用类初始化方法<clinit>，给静态成员变量赋初始值。



# 编译器
在JVM中，编译器会将字节码文件编译成机器码。Java是解释与编译并存编程语言

- 混合模式：-Xmixed
- 解释模式：-Xint
- 编译模式：-Xcomp，类文件多时启动慢
- 检测热点代码阈值：-XX:ComplieThreshold，默认值10000