# Java NIO Server
A Java NIO Server using non-blocking IO all the way through.

Note:
This is NOT intended for reuse "as is". This is an example of how you could design a Java NIO server yourself.
The design is explained in this tutorial:

[Java NIO Non-blocking Server](http://tutorials.jenkov.com/java-nio/non-blocking-server.html)


Because this is an example app - this project will NOT accept feature requests. If there are any obvious bugs in the code, I can fix those, but apart from that, the code has to stay as it is.

By the way, I am working a real, usable non-blocking client-server API called "Nanosai Net Ops" - based on the designs of this project.
That project contains both a client and a server, and you can use both client and server using both blocking and non-blocking methods,
and switch between the two modes as you see fit. You can find Net Ops here:

[https://github.com/nanosai/net-ops-java](https://github.com/nanosai/net-ops-java)

I have been able to "echo" around 200.000 messages per second with Net Ops (in early versions), coming from 3 clients running on the same machine, against a single-threaded server - on a Quad core CPU.

Net Ops has several smaller improvements in the design and functionality over the server you see in this project, so if you really want
to study a more robust non-blocking IO client / server design, look at Net Ops too.



## SocketProcessor

**引用的对象**

socketQueue

socket （包含通道，消息读取器）

readBuffer/writeBuffer  （内存）

messageReaderFactory

messageProcessor 

writeProxy  （从写缓存区获取消息，然后把它放入到队列） 

**核心方法**

 com.jenkov.nioserver.SocketProcessor#takeNewSockets （为所有socket 初始化，并向选择器注册读事件）

* （从队列获取新连接），
* 对连接属性进行赋值（包括nextSocketId, messageReader, messageWriter）
* 给客户端通道在选择器上注册读事件，以socket 对象作为附件（在通道可读，可以通过selectionKey 获取到）

com.jenkov.nioserver.SocketProcessor#readFromSockets  （**从准备就绪的通道读取数据**）

* com.jenkov.nioserver.SocketProcessor#readFromSocket 
  * 从SelectionKey 中获取Socket 对象
  * 使用socket.messageReader.read（socket,this.readByteBuffer） 把通道的字节读取到buffer中，默认为1M， 如果超过1M 如何处理 
  * 读取完成之后，就是获取消息，然后调用MessageProcessor.process 处理消息。
  * socket.endOfStreamReached == true 移除socket  ()



com.jenkov.nioserver.SocketProcessor#writeToSockets



## MessageBuffer

> 把内存逻辑上切换为三块（小，中，大）

**核心方法**

* Message getMessage() （获取消息对象，返回一块可用的内存）
* expandMessage（扩容消息，什么情况下会扩容？ 客户端写入的字节大于message 容量 ）

## Message

> 消息对象 

**核心方法**

* com.jenkov.nioserver.Message#writeToMessage(java.nio.ByteBuffer) 把byteBuffer 消息写入到内存
* com.jenkov.nioserver.Message#writePartialMessageToMessage  写入部分消息 



## MessageReader

> 实现类 HttpMessageReader ， 基于http 协议设计

包含三个方法 

* init(MessageBuffer readMessageBuffer)
  * 持久读共享内存引用
  * 获取下一个消息以及设置默认请求头信息 
* read(Socket socket, ByteBuffer byteBuffer)
* `List<Message> getMessages()`





## WriteProxy 

> 具备两个属性：
>
> messageBuffer  写缓冲
>
> writeQueue 写队列
>
> 作用：就是从写缓冲区获取一个buffer， 然后把响应消息写入到缓存中，在入队（通道写缓冲有数据）







## Http 协议

> http 的首行记录请求方法 和 协议版本
>
> 从第二行到 空行为请求头信息
>
> 从空行往后是body 内容。 

```http

POST / HTTP/1.1
Host: localhost:9999
User-Agent: curl/7.57.0
Accept: */*
Content-Length: 11
Content-Type: application/x-www-form-urlencoded

hello=world
```





## 整体流程梳理

1. 从socket队列中获取进入的连接
2. 为连接初始化 消息读处理器， 消息写处理器，以及设置非阻塞
3. 注册可读事件，读选择器轮询到读继续事件
4. 处理读就绪事件，消息读取器处理字节为一个完整消息（消息存储在内存中） 
5. MessProcessor 处理完整消息 （目前仅默认返回固定字节） 把响应数据入队到
  消息写入器队列中（outboundMessageQueue） 
6. MessageWriter 处理outboundMessageQueue 中消息
7. 注册可写事件 
8. 关闭已经响应过的sockets 
9. 选择写就绪sockets， 数据响应客户端









## 一个网络IO 框架需要考虑

线程模型 
如何读取一个完整的消息  （反序列化）
如何写入消息 （序列化）
如果解析不同的协议（支持相应协议特性，长连接，心跳，压缩等等）
内存的设计（考虑支持百万连接）







# 疑问点



**什么情况 endOfStreamReached  为true?** 
endOfStreamReached  socketChannel 中数据被读取完了。 



**com.jenkov.nioserver.Socket#read  如何读取?** 

（目前最多读取1M）socketChannel.read(byteBuffer)，把通道中数据读取到byteBuffer中， 

**读取到缓存区中数据如何处理** ? 
把bytebuffer 中数据写入到messageBuffer （一个字节数组的内存）
然后开始解析http 请求， 
如果解析成功。 
从messageBuffer 申请下一个空闲块。 



**nextMessage 什么时候获取下一个**  ？ 



**这两个集合做什么的 ？** 
emptyToNonEmptySockets   存放待注册写事件
nonEmptyToEmptySockets  存放已经写入完成的sockets，等待关闭 



> 小知识 
>
> **UTF-8 包含ascii码**
>
> UTF-8 编码把一个Unicode 字符根据数据大小编码成1-6 个字节，常用的英文被编码成1个字节， 汉字通常是3个字节。 
>
> CRLF 
>
> CR  （Carriage Return)  回车符， 用于将鼠标移动到行首，并不前进至下一行 
>
> LF  （Line Feed） 换行符 （\n） 