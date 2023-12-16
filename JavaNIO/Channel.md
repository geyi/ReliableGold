Java NIO的通道类似流，但又有些不同：既可以从通道中读取数据，又可以写数据到通道。但流的读写通常是单向的。通道中的数据总是要先读到一个Buffer，或者从一个Buffer中写入。正如上篇所说，数据总是从通道读取到缓冲区或者从缓冲区写入到通道。如下图所示：

![ChannelBuffer](../image/Java/JavaNIO/ChannelBuffer.png)

# Channel的实现
这些是Java NIO中最重要的通道的实现：
- FileChannel
- DatagramChannel
- SocketChannel
- ServerSocketChannel

FileChannel从文件中读写数据。DatagramChannel能通过UDP读写网络中的数据。SocketChannel能通过TCP读写网络中的数据。ServerSocketChannel可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。

# 基本的Channel示例
下面是一个使用FileChannel读取数据到Buffer中的示例：
```java
package com.airing.nio;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @Title: FileChannelTest.java
 * @Description:
 * @author GEYI
 * @date 2016年6月22日 下午4:23:49
 * @version V2.0
 */
public class FileChannelTest {
    public static void main(String[] args) {
        try {
            File file = new File("/Users/GEYI/tmp/test.txt");
            RandomAccessFile randomAccessFile = new RandomAccessFile(file, "rw");
            FileChannel fileChannel = randomAccessFile.getChannel();

            ByteBuffer byteBuffer = ByteBuffer.allocateDirect(10);

            System.out.println(byteBuffer.isDirect());

            int position = fileChannel.read(byteBuffer);

            File saveFile = new File("/Users/GEYI/tmp/test1.txt");
            FileOutputStream fileOutputStream = new FileOutputStream(saveFile);

            while (position != -1) {
                System.out.println("Read: " + position);
                // 使缓冲区为一系列新的通道读取或相对放置操作做好准备：它将限制设置为当前位置，然后将位置设置为 0
                byteBuffer.flip();
                while (byteBuffer.hasRemaining()) {
                    System.out.println("limit: " + byteBuffer.limit());
                    System.out.println("position: " + byteBuffer.position());
                    byte bufferData = byteBuffer.get();
                    System.out.println(bufferData);
                    byte[] b = new byte[1];
                    b[0] = bufferData;
                    fileOutputStream.write(b);
                }
                System.out.println("/////" + byteBuffer.position());
                // 使缓冲区为一系列新的通道写入或相对获取操作做好准备：它将限制设置为容量大小，将位置设置为 0
                byteBuffer.clear();
                System.out.println("/////" + byteBuffer.position());
                position = fileChannel.read(byteBuffer);
                System.out.println("/////" + byteBuffer.position());
            }
            fileOutputStream.close();
            randomAccessFile.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```