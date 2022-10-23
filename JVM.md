# Class File Format
IDEA插件
- BinEd
- jclasslib Bytecode viewer

# 类加载
## Loading
当我们使用一个ClassLoader加载一个类时，先会在当前类加载器中寻找是否已经加载过此类，有则直接返回。如果没有，则会委托父加载器去加载。父加载器同样先在自己的加载器中寻找是否已加载过此类，有则直接返回，如果也没有则继续委托给自己的父加载器去加载。直到最顶层的类加载器BoostrapClassLoader，如果仍然没有找到指定的类，BootstrapClassLoad先会尝试加载，如果无法加载，则会委托给子加载器去加载，如果仍然不行则继续向下委托，即，双亲委派。

类加载时会依据给定的二进制字节流，定义Java类，然后创建代表这个类或者接口的java.lang.Class对象。
1. 如果没有找到Java类或接口的二进制表示，就会抛出NoClassDefFound。
2. 此外，类加载阶段会对类的格式进行语法检查，如果有错，则会抛出ClassFormatError或UnsupportedClassVersionError。
3. Java类加载前，HotSpot VM必须先加载它的所有超类和超接口。如果类的继承层次有错，例如Java类是它自己的超类或超接口（类层次递归），HotSpot VM则会抛出ClassCircularityError。
4. 如果所引用的超接口本身并不是接口，或者直接超类实际上是接口，HotSpot VM则会抛出IncompatibleClassChangeError。

### 自定义类加载器
自定义ClassLoader，只需要重写findClass方法。（class文件二进制数据被加载进内存，然后通过defineClass方法定义一个Class类型的对象）
重写loadClass方法可以打破双亲委派机制。

## Linking
- Verification [ˌvɛrəfəˈkeɪʃən] ：验证文件是否符合JVM规范。例如，检查类文件的语义、常量池符号以及类型。如果检查有错，就会抛出VerifyError。
- Preparation [ˌprepəˈreɪʃn] ：静态成员变量（不包括实例变量）赋标椎默认值。标准默认值的意思是不同数据类型的零值（0，0.0，null，false）。特殊情况，如果变量被final关键字修饰，那么在该阶段变量将会被赋初始值。
- Resolution：解析符号引用。将类、方法、属性等符号引用替换为直接引用。虚拟机将常量池内的符号引用替换为直接引用的过程，也就是得到类、方法或者字段在内存中的指针或者偏移量。

> 标椎默认值，如int的标准默认为0，所以对于public static int i = 123，准备阶段将其初始化为0而不是123，i = 123的赋值操作在类构造器<clinit>()中。这有例外，对于public static final int i = 123，编译时会为i在字段属性表中生成ConstantValue，从而在准备阶段就被初始化成123。

## Initializing
调用类初始化方法<clinit>，给静态成员变量赋初始值。

> 出于性能优化的考虑，通常直到类初始化时HotSpot VM才会加载和链接类。这意味着，类A引用类B，加载A不一定导致加载B（除非类B需要验证）。

## Using
使用

## Unloading
卸载

# 编译器
在JVM中，编译器会将字节码文件编译成机器码。Java是解释与编译并存的编程语言

- 混合模式：-Xmixed
- 解释模式：-Xint
- 编译模式：-Xcomp，类文件多时启动慢
- 检测热点代码阈值：-XX:ComplieThreshold，默认值10000

# JVM垃圾回收日志配置参数
V1：`JAVA_OPTS="-Xms1280M -Xmx1280M -Dfile.encoding=UTF-8 -Duser.timezone=Asia/Kolkata -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/app/fts_im1/logs/gc.log"`

V2：`-Xloggc:/opt/xxx/logs/xxx-xxx-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCCause`

# JVM调优
通用法则之一：将Java堆的初始值-Xms和最大值-Xmx设置为老年代活跃数据大小的3-4倍。

通用法则之二：永久代的初始值-XX:PermSize及最大值-XX:MaxPermSize应该是永久代活跃数据的1.2-1.5倍。

补充法则：新生代空间应该为老年代空间活跃数据的1-1.5倍。

## 新生代
如果观测到YGC的平均持续时间大于应用程序的延迟性要求，可以适当减小新生代空间的大小，之后再运行测试，收集GC统计数据之后再次评估数据。如果观测到的YGC频率大于应用程序的延迟性要求（发生的太频繁），增大新生代空间，之后再运行测试，收集GC统计数据之后再次评估数据。（调整新生代空间大小时，尽量保持老年代空间大小恒定）

调整新生代大小时，需要谨记下面几个准则：
- 老年代空间大小不应该小于活跃数据大小的1.5倍。
- 新生代空间至少应该为Java堆大小的10%，通过-Xmn和-XX:NewRation可以设定该值。
- 增大Java堆大小时，需要注意不要超过JVM可用的物理内存。（使用交换内存反而造成性能降低）

### Survivor
减少Eden空间会导致更频繁的YGC，垃圾收集发送的频率越高，对象老化的速度就越快。

-XX:+PrintTenuringDistribution：输出每次YGC时晋升分布的情况

通常情况下，观察到新的晋升阈值持续小于最大晋升阈值，或者观察到Survivor空间大小小于总的存活对象大小（即对象年龄最后最后列的值）都表明Survivor空间过小。

通常情况下，即使在Survivor空间之间多次复制对象也比匆匆将对象提升到老年代要好。

## 老年代
老年代的空间占用情况可以通过YGC之后Java堆的占用情况 减去 同一次YGC后新生代的空间占用得到

提升率 = 每次YGC老年代空间占用增量 / YGC频率，例如：20MB/次 / 2s/次 = 10MB/s

得到 提升率 后，填充老年代可用空间的时间 = 可用空间大小 / 提升率，例如：
老年代可用空间大小为2048MB，提升率为10MB/S，占用老年代多需要的时间则为 2048 / 10 = 204.8S，大约是4分钟左右。

如果预期或观测到FGC的频率已经远远不能达到应用程序的最差FGC频率要求，就应该增大老年代空间的大小。（增加老年代空间的大小时注意保持新生代空间大小恒定）

## CMS
从Throughput迁移到CMS时，如果发生新生代至老年代的对象提升，可能会经历较长的YGC持续时间，这是由于对象提升到老年代变得更慢了。因为CMS在老年代空间从空闲列表中分配内存。与之相反，Throughput收集器只需要在线程本地分配的提升缓存中移动指针即可。

1. 内存分配效率降低
2. 吞吐量降低

使用CMS时，如果老年代空间用尽，就会触发一个单线程STW压缩式的垃圾收集。

从Throughput收集器迁移到CMS收集器时需要遵循的一个通用原则是，将老年代空间增大20%-30%。

CMS调优面临的问题：
1. 对象从新生代提升至老年代的速率。
2. 并行老年代垃圾收集线程回收空间的速率。
3. 碎片化。

解决碎片化的方案：
1. 压缩老年代空间。单STW压缩式GC耗时较长，应该尽量避免。
2. 让老年代的空间大到足以避免由堆内存碎片引起的STW压缩。
3. 减少对象从新生代提升至老年代的比率，即“YGC回收原则”。