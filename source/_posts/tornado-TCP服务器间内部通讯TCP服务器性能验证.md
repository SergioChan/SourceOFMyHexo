title: tornado-TCP服务器间内部通讯TCP服务器性能验证
date: 2015-07-19 09:08:40
categories: Python学习笔记
tags: [tornado, TCP]
---

> 本文附带项目Github仓库地址，随手star是个好习惯：
> [https://github.com/SergioChan/tornado-TCP](https://github.com/SergioChan/tornado-TCP)



# 为什么要有tornado-TCP
在实际的业务场景中，当一个系统复杂到一定程度后，很多服务都需要被独立地分割出来，部署到独立的服务器上。例如日志服务，图像服务，短信服务和一些数据分析服务这些可能会被许多功能模块共用的且对服务器性能有一定消耗的服务。当功能划分后，各服务器之间就需要通过内部调用连接在一起，通常来说简便的做法就是通过HTTP请求，这样外部和内部访问的服务器都是通用的，对于开发，维护和部署来说是省去了不少功夫。但是，一方面，有一些HTTP Server可能会同时处理内部和外部请求，如果从其他模块发来的内部请求过多，占用了HTTP Server的处理资源，外部客户端的请求可能会因此变慢，因此对于一个想要拥有更高可靠性和稳定性的大型系统来说，将内部调用和外部调用从逻辑上分离开是一个比较好的优化手段。另一方面，虽然在这种场景下HTTP请求的性能不会差到哪里去，在拥有局域网的情况下，这种内部调用的处理速度会相当快，它的连接也会很快的被释放和刷新，但是由于HTTP毕竟是TCP上的应用层，TCP省去了一些HTTP Header的传输和消耗，因此一定意义上采用更低级的TCP来作为内部调用的传输手段，既是一种较为标准和专业化的方法，也是一种更加优化的方法。  
我们先要理解HTTP连接和TCP连接的区别。首先，HTTP连接是基于TCP连接的。HTTP（HyperText Transport Protocol）是超文本传输协议的缩写。 HTTP协议采用了请求/响应模型。客户端向服务器发送一个请求，其中，请求头包含请求的方法、URL、协议版本、以及包含请求修饰符、客户信息和内容的类似于MIME的消息结构。服务器以一个状态行作为响应，响应的内容包括消息协议的版本，成功或者错误编码加上包含服务器信息、实体元信息以及可能的实体内容。它的最显著的特点是客户端发送的每次请求都需要服务器回送响应，在请求结束后，客户端会主动释放连接。从建立连接到关闭连接的过程称为“一次连接”。这个特点也使得HTTP协议成为了包括移动和web浏览器在内的客户端和服务器端通信的标准途径，因为大部分客户端和服务器的交互都是一次性的，例如一次读和一次写。当每次交互都是原子的时候，在没有特别需求的场景维持长连接对于资源是一种浪费，因此大部分客户端和服务器通信都采用了HTTP协议。虽然现在已经有一些HTTP长连接的实现，但它的机制其实也是基于HTTP协议的，通过类似心跳的模式保持HTTP连接不会被释放。  
TCP提供一种面向连接的、可靠的字节流服务。面向连接意味着两个使用TCP的应用，通常是一个客户和一个服务器。在彼此交换数据包之前必须先建立一个TCP连接。在一个TCP连接中，仅有两方进行彼此通信。在普遍的场景中，TCP长连接常用于客户端推送、即时通信的实现。当然，在很多情况下，TCP也被用来作为服务器内部服务调用的实现。像日志服务这种公用的服务模块，整个系统对它的调用是十分频繁的，因此采用“一次连接”模式的HTTP请求和采用保持连接的TCP请求的区别十分明显，采用TCP会使系统省下大量的资源去重新建立和释放连接。

# 实际结果

对于服务器内部通信采用HTTP协议和TCP协议的性能表现差异，我用对比实验的方式来验证结果。首先，我模拟了传统的HTTP Server的环境，采用了Django作为服务器，并实现了一个基于HTTP协议的内部调用的协议。同时，我将tornado-TCP也部署在相同机器上，在Django上同样实现了一个基于TCP协议的内部调用的协议。然后通过压力测试，得出了以下结果：
## 每秒并发的流量测试：
基于HTTP的内部调用的协议请求：  
Used 1.00223088264 s for requests, success count is: 289  
基于TCP的内部调用的协议请求：  
Used 1.00016713142 s for requests, success count is: 633  


## 一定数量并发请求的处理速度：

基于HTTP的内部调用的协议请求：  
Used 47.3748078346 s for 10,000 requests, success count is: 9998  
Used 281.623967171 s for 50,000 requests, success count is: 49999  
基于TCP的内部调用的协议请求：  
Used 14.8007540703 s for 10,000 requests, success count is: 9999  
Used 123.114969015 s for 50,000 requests, success count is: 49999   
以上的测试结果均基于相同的Django环境，由于测试的时候使用的是单进程的Django自带的HTTP Server，因此TCP Server也采用了单进程模式。如果切换成多进程模式，采用的原理和tornado相同。也就是说，当基于Django的HTTP Server以单进程模式运行的时候，其处理速度大概是相同条件的tornado-TCP服务器的一半以上。  
这是这三个服务器的进程情况：

```
501 68455  1492   0 10:11上午 ??         0:08.05 /usr/bin/python /Users/useruser/tornado-TCP/Server/Manage.py
501 68465 68433   0 10:12上午 ttys001    0:00.63 /usr/bin/python manage.py runserver 127.0.0.1:9555
501 68468 68465   0 10:12上午 ttys001    0:08.13 /usr/bin/python manage.py runserver 127.0.0.1:9555
501 68475 68472   0 10:12上午 ttys002    0:00.43 /usr/bin/python manage.py runserver 127.0.0.1:9556
501 68478 68475   0 10:12上午 ttys002    0:43.47 /usr/bin/python manage.py runserver 127.0.0.1:9556
```
其中9555端口运行的是内部调用的HTTP Server，9556端口运行的是主测试服务器，TCP服务器运行在8889端口上。

# 框架介绍

## tornado-TCP framework
tornado-TCP的框架可以由以上的架构图来表示。和传统的HTTP Server相比，它相当于是在不同的Server之间建立了一个长连接，而这个连接的主体是一个IOStream，由Connection类来保持监听和字节流的读取。其余模块都是参考传统HTTP Server的架构来添加的。  
任意其他连接到tornado-TCP的服务器实际上都是一个客户端。每个客户端都需要自己维护一个和tornado-TCP服务器的TCP Socket连接。如果这种内部调用在可预见的范围内十分的频繁，这个连接最好在全局建立和维护，随着服务器的启动初始化。保持IOStream的连接可以使得服务器之间的局域网通信比传统的HTTP请求更快。

## Connection


Any other server that was connected to this tornado-TCP server will be represented as a client-server. Each client-server will maintain a TCP connection with tornado-TCP server. If the connection will be used very frequently, it’s better not to close it. Keeping an IOStream for the connection will make the communication in the server-side Local Area Network faster than typical HTTP request.  
Connection will read the request from IOStream and handle the requests. When a request handling is over, the Connection will automatically read the stream to get the next request. As the source code shown below:

```Python
def handle_request(self, data):
    tmp_body = data[:-1]

    request = Request(address=self._address, Body=tmp_body)
    handler = urls.Handler_mapping.get(request.cmdid)

    handler_instance = handler()

    if isinstance(handler_instance,BaseHandler):
        try:
            handler_instance.process(request=request)
            self._stream.write(handler_instance.res)
        except Exception as e:
            baseLogger.error(e.message)

self.read_request()
```

## Handler


Handler is the class type for processing the request object. You can sub-class your own Handler from BaseHandler to implement custom processing method. This is the sample of TestHandler:

```Python
@urls.handler(constant.TEST_CMDID)
class TestHandler(BaseHandler):

    def process(self,request):
        if isinstance(request, Request):
            print request.params
        else:
            raise TypeError
        
```

Notice the decorator @urls.handler which is used to add mapping between cmdId and Handler. The definition of this decorator is in the urls.py. Each custom Handler should decorated by this decorator.

## Request


Request is the basic object type in the IOStream. Each Request is delimited by a delimiter ‘\n’.Request is transported in a serialization mode, using normal json type.   
To extend Request, define a subclass and there is no need to override anything.
Request has following parameters:  

* 1.address: simple ip address combined with port number representing the request origin server.
* 2.rawBody: raw content of the request body
* 3.cmdid: command id defines in Command
* 4.timestamp: the date when request was sent
* 5.params: a dict that storage all the data it takes

详细介绍和代码请前往[https://github.com/SergioChan/tornado-TCP](https://github.com/SergioChan/tornado-TCP)
欢迎关注和批评指导！