
1. 考虑客户端的运行环境，客户端需要运行在机器人上，机器人上的系统不同于普通的电脑，内存会比普通电脑小很多，另外机器人所处环境的网络条件也没有开发电脑上的网络条件好。为了性能考虑，需要使用机器人电脑上尽量少的空间，与外面电脑发送尽可能少的请求。为了满足上述需求，应用程序设计为将服务器端以sdk的形式放在电脑上，开启使用nodejs子进程运行。为了满足客户端与服务器端的双向数据交互，设计通信模块，该模块需要包含如下功能：

  (1) 建立连接。作为客户端主动与服务器端建立连接，当连接出错或者无法检测到心跳时需要重新连接。
 
  (2) 断开连接。两端都可以主动断开连接。
 
  (3) 发送数据。需要考虑发送的数据是给另一端响应的消息还是主动向另一端发送的消息，也需要标注主动给另一端发送的请求是否需要响应。

  (4) 接收数据。处理接收数据要考虑TCP粘包的问题，即另一端可能同时发送多条数据。（其实还需要考虑拆包的问题，即有可能一条消息的数据量很大导致分多次传递，但是当前业务场景不需要）。另外也需要考虑接受到的数据是否需要给请求端返回响应。
  
  (5) 心跳检测。当连接中断时程序会报错误信息，但是可能存在由于网络问题等导致连接没有中断但是无法传输数据的情况，为了保证数据可以准时地到达，为程序设置了心跳检测机制，当确定当前连接无法正常传输数据时，程序会主动将连接断开然后重新建立连接。

2. 模块设计

 一般来说，要满足双向通信有下面两种方式：

（1）	建立两条通信通道，两端分别有自己的服务器，各自处理各自的请求与响应。

（2）	建立一条通信通道。可以维持一个tcp长连接，或者使用websocket。

Websocket是基于http协议，需要在机器人上运行一个http server。尽可能的减少在机器人上启动的服务，尽可能的节约资源，采用的建立TCP长连接的方式。实现的方式是使用NodeJs的net模块。具体的模块设计如下。

2.1 建立连接

  客户端作为client主动与server建立连接，nodejs中提供了net.connect()方法与服务器端建立连接。提供了“error”事件进行错误监听。当监听到“error”事件时，调用socket.destroy()方法删除连接，然后重新与服务器建立连接，连接的策略是前5次检测到错误之后马上进行重连，之后间隔5s重连一次。

2.2 断开连接

  使用nodejs提供的client.end()方法可以断开与服务器的连接，当关闭应用窗口时应调用此方法。

2.3 发送数据

发送数据分三种情况

（1）	发送的是另一端请求的数据的结果

（2）	发送的是不需要回复的消息

（3）	发送的是需要回复消息的请求

对于第一种情况，请求端发来的请求到对应的处理程序中执行，执行之后可以直接将执行的结果发送给请求端。此时发送的消息应携带请求的id，发送消息的类型（响应）和结果信息。

对于第二种情况，直接发送消息即可。

对于第三章情况，采用的方案是发送消息的同时，将消息的id和对应的响应处理程序存放到一个map中，同时设置一个定时器，当超过定时器设置的时间没有收到回复消息时，认为请求超时，提示用户请求超时，同时删除map中的消息id与响应处理程序。当收到回复的消息时，到map中查找响应的处理程序并执行。
为了满足上面的要求，客户端与服务器端定义数据传递格式如下：

（1）	消息的类型。请求还是响应。

（2）	消息id。设置消息id主要是为了将请求与响应相关联，如果发送的消息不需要得到响应信息，则无需携带消息id，或者可以将消息id设为-1。

（3）	消息处理器。发送的消息应由另一端的哪个方法处理。

（4）	消息的长度。为了确定收到的消息是不是完整的，有没有丢失。

（5）	结尾标识。每一条消息最后都应该携带一个结尾的标识，区分一条消息的结束。

2.4 接收数据

接收到的数据会有下面几种类型：

（1）	接收到的是不需要回复的指令。

（2）	接收到的是需要回复的请求。

（3）	接收到的是之前发出的请求的响应。

对于第一种情况，只需找到对应handler的处理程序并执行即可。

对于第二种情况，请求端发来的请求应该携带handler和id两种信息，在被请求的handler对应该的处理程序中是存在返回值的，请求端发来的请求到对应的处理程序中执行，执行之后可以直接将执行的结果发送给请求端。此时发送的消息应携带请求的id。为了方便使用，对方法进行包装，指定一个特定的handler专门处理需要返回消息的请求，在里面调用请求中指定的handler，得到返回值，并发送给请求端，这样写handler的代码时，如果需要向请求端返回数据，则只需要将返回的结果写到return中即可。代码如下所示：

```
handlerMap.actionWithResponse = function (command, content, id) {
    let result = {}
    if (command) {
      result = handlerMap[command](content)
    }
    context.$remoteApi.postResponse({
      message: result,
      id: id
    })
  }
```

其中command为请求端指定的处理请求的真正handler，content为请求内容，id为请求的id。其实本不需要写个额外的handler，但是在本项目中，处理消息的逻辑写在渲染进程中，而接受消息的方法写在主进程中，主进程调用渲染进程的方法时无法得到在渲染进程的方法return的返回值，所以发送返回值的逻辑必须要写到渲染进程中，所以需要一个额外的handler。
对于第三种情况，当收到的消息类型是响应时，需要根据消息的id，到保存消息id与对应处理程序的map中找到对应的处理程序，如果找到则执行，如果找不到则不做处理，一般情况下只有当请求超时才会出现找不到id对应的处理程序的情况。

2.5 心跳检测

心跳检测的设计方案是客户端每5s向服务器端发送一次心跳包，之后服务器端向客户端返回一个心跳包。如果客户端连续三次没有收到回复消息，则认为心跳丢失。实现的方式是连接成功之后开始发送数据，并设置一个全局变量sendUrgentTime记录当前连续没有收到回复的心跳包数量，发送一次sendUrgentTime加一，收到消息sendUrgentTime置为0，因为能收到消息便可以认为当前通道可用，当sendUrgentTime加到3时，断开当前连接，重新建立新的连接。


