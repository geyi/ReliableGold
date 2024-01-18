Java NIO 由以下几个核心部分组成：
- Channels
- Buffers
- Selectors

虽然 Java NIO 还有其他一些类和组件（如 Pipe 和 FileLock），但Channel，Buffer和Selector构成了Java NIO的核心API。

# Channel 和 Buffer
BIO 以流的方式处理数据，而 NIO 以缓冲区（也被叫做块）的方式处理数据。BIO 基于字符流或者字节流进行操作，而 NIO 基于 Channel 和 Buffer 进行操作，数据总是从通道读取到缓冲区或者从缓冲区写入到通道。如下图所示：

![ChannelBuffer](../image/Java/JavaNIO/ChannelBuffer.png)

Channel是一个顶层接口，它的常用子类有：
- FileChannel：用于文件读写
- DatagramChannel：用于UDP数据包收发
- SocketChannel：用于客户端TCP数据包收发
- ServerSocketChannel：用于服务端TCP数据包收发

Buffer 是一个顶层抽象类，它的常用子类有（前缀表示该 Buffer 可以存储哪种类型的数据）：
- ByteBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer
- CharBuffer

这些Buffer覆盖了你能通过IO发送的基本数据类型：byte, short, int, long, float, double和char。Java NIO 还有个**MappedByteBuffer**，用于表示内存映射文件。

# Selector
Selector（选择器）是实现 IO 多路复用的关键，多个 Channel 注册到某个 Selector 上，当 Channel 上有事件发生时，Selector 就会取得事件然后调用线程去处理事件。也就是说只有当连接上真正有读写等事件发生时，线程才会去进行读写等操作，这就不必为每个连接都创建一个线程，一个线程可以应对多个连接。以下是在一个单线程中使用一个Selector处理3个Channel的图示：

![Selector](../image/Java/JavaNIO/Selector.png)

要使用Selector，需要向Selector注册Channel，然后调用它的select()方法。这个方法会一直阻塞到某个注册的通道有事件就绪（比如收到连接请求，数据到达等等）。一旦这个方法返回，线程就可以处理这些事件。
