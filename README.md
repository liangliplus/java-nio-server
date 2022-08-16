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
emptyToNonEmptySockets 
nonEmptyToEmptySockets 