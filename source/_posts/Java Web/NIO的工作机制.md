---
title: NIO工作机制
date: 2016-05-18
categories: Java Web
tags: 
- Java
- NIO 
---

#	NIO的工作方式

##	NIO工作机制
NIO引入了Channel、Buffer和Selector，Channel是高速通道，Buffer是高速通道上的具体单位，Selector是调度器。当我们调用write()往SendQ写数据时，当一次写的数据超过SendQ长度时需要按照SendQ的长度进行分割，在这个过程中需要将用户空间数据和内核地址空间进行切换，而这个切换不是你可以控制的，但在Buffer中我们可以控制Buffer的容量、是否扩容以及如何扩容。

调用Selector的静态工厂创建一个选择器，创建一个服务端的Channel，绑定到一个Socket对象，并把这个通信信道注册到选择器上，把这个通信信道设置为非阻塞模式。然后就可以调用Selector的selectedKeys方法来检查已经注册在这个选择器上的所有通信信道是否有需要的事件发生，如果与某个事件发生，将会返回所有的SelectionKey，通过这个对象的Channel方法就可以就可以取得这个通信信道对象，从而读取通信的数据，而这里读取的数据是Buffer，这个Buffer是我们可以控制的缓冲器。

```	java
public void selector() throws IOException {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        Selector selector = Selector.open();
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.configureBlocking(false); //设置为非阻塞方式
        ssc.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            Set selectedKeys = selector.selectedKeys();
            Iterator it = selectedKeys.iterator();
            while (it.hasNext()) {
                SelectionKey key = (SelectionKey) it.next();
                if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {
                    ServerSocketChannel ssChannel = (ServerSocketChannel) key.channel();
                    SocketChannel sc = ssChannel.accept(); //接受到服务端请求
                    sc.configureBlocking(false);
                    sc.register(selector, SelectionKey.OP_READ);
                    it.remove();
                } else if ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
                    SocketChannel sc = (SocketChannel) key.channel();
                    while (true) {
                        buffer.clear();
                        int n = sc.read(buffer);
                        if (n <= 0) {
                            break;
                        }
                        buffer.flip();
                    }
                    it.remove();
                }
            }
        }
    }
```

在上面这段程序中，将Server端的监听连接请求的事件和处理请求的事件放在一个线程中，但是在事件应用中，我们通常会把它们放在两个线程中：一个线程专门负责监听客户端的连接请求，而且是以阻塞方式运行的；另外一个线程专门负责处理请求，这个专门处理请求的线程才会真正采用NIO的方式，像Web服务器Tomcat和Jetty都是使用这个处理方式。

Selector可以同时监听一组通信信道(Channel)上的IO状态，前提是这个Selector已经注册到这些通信信道中。选择器Selector可以调用select()方法检查已经注册的通信信道上IO是否已经准备好，如果没有至少一个信道IO状态有变化，那么select方法会阻塞等待或在超时时间后返回0。如果有多个信道有数据，那么将会把这些数据分配到对应的数据Buffer中。所以关键的地方是，有一个线程来处理所有连接的数据交互，每个连接的数据交互都不是阻塞方式，所以可以同时处理大量的连续请求。

##	Buffer的工作方式
把Buffer简单理解为一组基本数据类型的元素列表，它通过几个变量来保存这个数据的当前位置状态，也就是有4个索引，如下所示：   
		
|索引|说明|
|:---:|:---:| 
|capacity|缓冲区数组的总长度|
|position|下一个要操作的数据元素位置|
|limit|缓冲区数组中不可操作的下一个元素的位置，limit<=capacity|
|mark|记录当前position的前一根位置或默认是0|

通过Channel获取的IO数据首先要经过OS的Socket缓冲区，再将数据复制到Buffer中，这个OS缓冲区就是底层的TCP所关联的RecvQ或者SendQ队列，从OS缓冲区到用户缓冲区复制数据比较耗性能，Buffer提供了另外一种直接操作OS缓冲区的方式，即ByteBuffer.allocateDirector(size)，这个方法返回的DirectByteBuffer就是与底层存储空间关联的缓冲区，它通过Native代码操作非JVM堆的内存空间。每次创建或者释放的时候都调用一次System.gc()。

DirectByteBuffer和Non-DirectBuffer对比
				
				
| |HeapByteBuffer|DirectByteBuffer|
|:---:|:---:|:---:| 
|存储位置|Java Heap中|Native内存中
|I/O|需要在用户地址空间和OS内核地址空间交换数据|不需复制|
|内存管理|Java GC回收，创建和回收开销少|通过调用System.gc()要释放掉Java对象引用的DirectByteBuffer内存空间，如果Java对象长时间持有引用会导致Native内存泄露，创建和回收内存开销较大|
|适用场景|并发连接数少于1000，I/O操作较少时比较合适|数据量比较大，生命周期比较长的情况下比较合适|


##	NIO的数据访问方式
NIO提供了比传统文件访问方式更好的方法，NIO有2个优化方法：一个是FileChannel.transferTo、FileChannel.transferFrom;另一个是FileChannel.map。

1.	FileChannel.transferXXX  
与传统的访问文件方式相比，可以减少数据从内核到用户空间的复制，数据直接在内核空间中移动，在Linux中使用sendfile系统调用。

2.	FileChannel.map
将文件按照一定大小块映射为内存区域，当程序访问这个内存区域时将直接操作这个文件数据，这种方式省去了数据从内核空间向用户空间复制的损耗。这种方式适合对大文件的只读性操作，如大文件的MD5校验。这种方式是和OS的底层I/O实现相关的，代码如下：

```	java
    public static void map() {
        int bufferSize = 1024;
        String fileName = "test.db";
        long fileLength = new File(fileName).length();
        int bufferCount = 1 + (int) (fileLength / bufferSize);
        MappedByteBuffer[] buffers = new MappedByteBuffer[bufferCount];
        long remaining = fileLength;
        for (int i = 0; i < bufferCount; i++) {
            RandomAccessFile file;
            try {
                file = new RandomAccessFile(fileName, "r");
                buffers[i] = file.getChannel().map(FileChannel.MapMode.READ_ONLY, i * bufferSize, (int) Math.min(remaining, bufferSize));
            } catch (Exception e) {

            }
            remaining -= bufferSize;
        }
    }
```
