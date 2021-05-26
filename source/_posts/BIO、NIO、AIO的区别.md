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



###### 场景

- 连接数较多
- 短连接

###### 构成

- Channel：全双工通道
- Buffer：数据缓冲区
- Selector：多路复用器

##### 异步非阻塞(AIO)

###### 场景

- 连接数较多
- 长连接



