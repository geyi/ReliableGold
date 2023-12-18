Buffer（缓冲区）用于与Channel进行交互。缓冲区本质上是一块可以写入数据，然后从中读取数据的内存。这块内存被包装成Buffer对象，并提供了一组方法，用来方便的访问该块内存。

# Buffer的基本用法
使用Buffer读写数据一般遵循以下四个步骤：
1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用clear()方法或者compact()方法

当向buffer写入数据时，buffer会记录下写了多少数据。一旦要读取数据，需要通过flip()方法将Buffer从写模式切换到读模式。在读模式下，可以读取之前写入到buffer的所有数据。

一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。有两种方式能清空缓冲区：调用clear()或compact()方法。

clear()方法会清空整个缓冲区。compact()方法只会清除已经读过的数据，任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。

# Buffer的capacity, position和limit
为了理解Buffer的工作原理，需要熟悉它的三个属性：
- capacity
- position
- limit

position和limit的含义取决于Buffer处在读模式还是写模式。不管Buffer处在什么模式，capacity的含义总是一样的。

这里有一个关于capacity，position和limit在读写模式中的说明，详细的解释在插图后面。

![](../image/Java/JavaNIO/Buffer.png)

## capacity
Buffer的capacity表示它所能容纳的最大数据量，即缓冲区的大小。
## position
Buffer的position表示当前操作的位置。在写入数据时，position表示下一个要写入的位置；在读取数据时，position表示下一个要读取的位置。初始时，position通常设置为0，随着读写操作的进行，position会自动**向前**移动。
## limit
Buffer的limit表示有效数据的范围。在写模式下，limit等于capacity，表示可以写入的最大数据量。而在读模式下，limit表示已经写入数据的末端位置，即可读取的最大数据范围。

# Buffer的类型
Java NIO 有以下Buffer类型
- ByteBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer
- CharBuffer
- MappedByteBuffer

这些Buffer类型代表了不同的数据类型。换句话说，就是可以通过char，short，int，long，float或double类型来操作缓冲区中的数据。

## Buffer的分配
要想获得一个Buffer对象首先要进行分配。 每一种Buffer类都有一个allocate方法。🌰：
```java
// 分配一个可存储48个字节的ByteBuffer
ByteBuffer buf = ByteBuffer.allocate(48);
// 分配一个可存储1024个字符的CharBuffer
CharBuffer buf = CharBuffer.allocate(1024);
```

## 向Buffer中写数据
写数据到Buffer有两种方式：
- 从Channel写到Buffer。
- 通过Buffer的put()方法写到Buffer。

```java
// 从Channel读取数据到Buffer
int position = channel.read(buf);
// 通过put方法写Buffer
buf.put(127);
```
put方法有很多版本，允许你以不同的方式把数据写入到Buffer中。例如，写到一个指定的位置，或者把一个字节数组写入到Buffer。更多Buffer实现的细节参考JavaDoc。

## flip()方法
flip方法将Buffer从写模式切换到读模式。调用flip()方法会将position设回0，并将limit设置成之前position的值。换句话说，position现在用于标记读的位置，limit表示之前写入了多少数据（现在能读取的最大数据范围）。

## 从Buffer中读取数据
从Buffer中读取数据有两种方式：
1. 从Buffer读取数据到Channel。
2. 使用get()方法从Buffer中读取数据。

```java
// 从Buffer写入数据到Channel
int position = channel.write(buf);
// 使用get()方法从Buffer中读取数据
byte data = buf.get();
```
get方法有很多版本，允许你以不同的方式从Buffer中读取数据。例如，从指定position读取，或者从Buffer中读取数据到字节数组。更多Buffer实现的细节参考JavaDoc。

## rewind()方法
Buffer.rewind()将position设回0，所以你可以重读Buffer中的所有数据。limit保持不变，仍然表示能从Buffer中读取多少数据。

## clear()与compact()方法
一旦读完Buffer中的数据，需要让Buffer准备好再次被写入。可以通过clear()或compact()方法来完成。

如果调用的是clear()方法，position将被设回0，limit被设置成capacity的值。换句话说，Buffer被“清空”了。但Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据。

如果Buffer中有一些未读的数据，调用clear()方法，数据将“被遗忘”，意味着不再有任何标记会告诉你哪些数据被读过，哪些还没有。

如果Buffer中仍有未读的数据，且后续还需要继续读取这些数据，但是此时想要先写些数据，那么使用compact()方法。compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素的后面。limit属性依然像clear()方法一样，设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据。

## mark()与reset()方法
通过调用Buffer.mark()方法，可以标记Buffer中的一个特定position。之后可以通过调用Buffer.reset()方法恢复到这个position。例如：
```java
buffer.mark();
buffer.reset();
```

## equals()与compareTo()方法
可以使用equals()和compareTo()方法比较两个Buffer。

### equals()
当满足下列条件时，表示两个Buffer相等（以ByteBuffer为例）：
1. 都属于ByteBuffer类型。
2. Buffer中从position到limit之前的元素个数相等。
3. Buffer中从position到limit之前的元素都相同。

如你所见，equals只是比较Buffer的一部分，不是每一个在它里面的元素都比较。

### compareTo()方法
compareTo()方法比较两个Buffer的剩余元素（byte、char等），如果满足下列条件，则认为一个Buffer“小于”另一个Buffer：
1. 第一个不相等的元素小于另一个Buffer中对应的元素。
2. 所有元素都相等，但第一个Buffer比另一个先耗尽（第一个Buffer的元素个数比另一个少）。

        
