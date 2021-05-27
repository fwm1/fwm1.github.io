---
title: BIO、NIO、AIO的区别
date: 2021-05-25 15:26:44
tags: [NIO]
categories: Java
---

##### 同步和异步

- 同步：一个流程中的每隔方法必须依次执行，具体到IO操作就是，线程触发IO操作并等待IO完成，或者需要轮询查看IO操作是否完成。
- 异步：调用方法立即返回，不需要等待方法结束就可以继续执行其他任务，具体到IO操作就是，线程触发IO操作后立即返回，当IO操作完成时会得到内核**通知**触发回调。

##### 阻塞和非阻塞

- 阻塞：单个线程内遇到同步等待后，一直在原地等待同步方法返回结果，具体到IO操作就是，当试图对FD或socket进行IO读写时，若当时没有内容可读或无法写入，线程就一直等待，知道有内容可读或可写。
- 非阻塞：单个线程内遇到同步等待后，不会原地等待，而是直接返回，具体到IO操作就是，当试图对DF或socket进行读写时，若当时没有内容可读或无法写入，读写函数立即返回，不会等待。

**阻塞与非阻塞指的的是当不能进行读写（网卡满时的写/网卡空的时候的读）的时候，I/O 操作立即返回还是阻塞；同步异步指的是，当数据已经ready的时候，读写操作是同步读还是异步读。**

##### 常见IO模型对比

所有的系统IO分为2个阶段：**等待IO就绪** 和 **实际IO操作**，举例来说，socket.read()分为等待网卡buffer有可读数据和buffer中的数据复制到用户空间。

- 如果是BIO，等待网卡IO就绪和复制数据到用户空间这两个步骤都是阻塞的；

- 如果是NIO，等待网卡IO就绪是非阻塞的，若网卡buffer中无可读数据，则直接返回；但复制数据到用户空间这个过程仍然是阻塞的，因此是同步非阻塞模型。

- 如果是AIO，这两个过程都是非阻塞的。

下图是几种常见的IO模型对比：

![image-20210527151101643](https://i.loli.net/2021/05/27/8mfnS3iXq67xlsw.png)



##### 同步阻塞(BIO)

###### 场景

- 连接数较小
- 连接数比较固定

###### demo

```java
public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress("0.0.0.0", 8888));
        Socket socket;
        while ((socket = serverSocket.accept()) != null) {
            final Socket client = socket;
            executorService.submit(() -> {
                try (InputStream is = client.getInputStream();
                     OutputStream os = client.getOutputStream();) {
                    byte[] bytes = new byte[1024];
                    int c;
                    while ((c = is.read(bytes)) != -1) {
                        os.write(bytes, 0, c);
                    }
                    os.flush();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }
    }
```

##### 同步非阻塞(NIO)

###### Buffer

- ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer...
- MappedByteBuffer(mmap)
- HeapByteBuffer
- DirectByteBuffer  通过FullGC回收内存

###### Channel

- FileChannel(文件)
- DatagramChannel(UDP)
- SocketChannel、ServerSocketChannel(TCP)

###### Selector

- selector.select() 是阻塞的，无论是通过操作系统的通知(epoll)还是不停地轮询(select，poll)
- selector.wakeup()解除阻塞在select上的线程，立即返回

###### 场景

- 连接数较多
- 短连接

###### 问题

- NIO依赖操作系统各自实现，JAVA原生的NIO编程复杂，不宜用；
- 建议使用成熟的NIO框架入Netty，解决了NIO的很多陷阱，屏蔽了不同操作系统的差异。

###### demo

```java
 public static void main(String[] args) throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.bind(new InetSocketAddress("0.0.0.0", 8888));
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            selector.select();
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                if (!key.isValid()) {
                    continue;
                }
                if (key.isAcceptable()) {
                    ServerSocketChannel serverChannel = ((ServerSocketChannel) key.channel());
                    SocketChannel clientChannel = serverChannel.accept();
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.wrap(new byte[1024]);
                    int read = clientChannel.read(buffer);
                    if (read == -1) {
                        key.cancel();
                        clientChannel.close();
                    } else {
                        buffer.flip();
                        clientChannel.write(buffer);
                    }
                }
            }
            iterator.remove();
        }
    }
```



###### 构成

- Channel：全双工通道
- Buffer：数据缓冲区
- Selector：多路复用器

##### 异步非阻塞(AIO)

###### 场景

- 连接数较多
- 长连接



*参考*：

https://zhuanlan.zhihu.com/p/23488863

https://developer.aliyun.com/article/726698





