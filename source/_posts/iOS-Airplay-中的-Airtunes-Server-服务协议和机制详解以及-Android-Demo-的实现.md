title: iOS Airplay 中的 Airtunes Server 服务协议和机制详解以及 Android Demo 的实现
date: 2016-08-29 10:03:22
categories: 细心写的技术博客


tags: [Airplay, Android, iOS]
---

在 Android 设备上搭建一个 Airplay Server 其实是一件很浩大的工程，因为这需要逆向苹果的 Airplay 协议流程啊格式啊什么的，万幸这件事情已经由许许多多国外的大神们帮我们做好了，因此我们只要基于他们逆向出来的 Airplay 协议来搭建一个服务就可以了。话虽如此，整个过程中的工作量和需要掌握的知识点还是非常非常多的。

在局域网中实现流媒体传输的主流协议有两种，一种是苹果封闭的 Airplay 协议，一种是 DLNA 。

> **D**IGITAL **L**IVING **N**ETWORK **A**LLIANCE 数字生活网络联盟，是索尼、英特尔、微软等发起的一套 PC、移动设备、消费电器之间互联互通的协议。它们的宗旨是“随时随地享受音乐、照片和视频”。据说苹果当时也是 DLNA 联盟的成员，而后来退出了并自立门户。

对于 iOS 系统来说，对用户最友好且体验最好的方式自然还是通过 Airplay 协议了（其实我是不太喜欢在每个单独的视频或者音乐播放器里面去找到 DLNA 或者 Airplay 的按钮然后切换模式，系统级的服务体验还是更好一些，因此我更倾向使用 Airplay）。因此在很多场景下，你需要让你的**安卓硬件**或者**设备**支持 Airplay 服务，本文就是通过一步步解释和分析这个基于 DroidPlay 改出的稳定可用的 Airtunes 服务，给大家展示一个比较清晰的 Airplay 中的 Airtunes 的机制和服务流程。

> 代码在 GitHub 上开放给大家学习和改动。地址在[这里](https://github.com/SergioChan/Android-Airplay-Server)。

## 首先，如何让 iOS 设备发现你

这是万事开头的第一步：你需要让自己的安卓设备出现在 iOS 设备 Airplay 的设备列表中。由于 Airplay 是基于局域网的，苹果设备会在当前局域网里搜寻支持 Airplay 服务的设备，因此在这里你就需要通过 mDNS 服务向局域网发送一个组播来让 iOS 设备能够在内网中发现你。在 Android 上你可以使用 **jmDNS** 库来实现这个功能:

```
final JmDNS jmDNS = JmDNS.create(addr, hostName + "-jmdns");
jmDNSInstances.add(jmDNS);

/* Publish RAOP service */
final ServiceInfo airTunesServiceInfo = ServiceInfo.create(
AIR_TUNES_SERVICE_TYPE,
hardwareAddressString + "@" + hostName,
getRtspPort(),
0 /* weight */, 0 /* priority */,
AIRTUNES_SERVICE_PROPERTIES
);
jmDNS.registerService(airTunesServiceInfo);
```

这个注册的服务类型和参数都是固定的，服务类型为 `_raop._tcp.local.`，参数列表如下:

```
"txtvers", "1",
"tp", "UDP",
"ch", "2",
"ss", "16",
"sr", "44100",
"pw", "false",
"sm", "false",
"sv", "false",
"ek", "1",
"et", "0,1",
"cn", "0,1",
"vn", "3"
```

通过注册上这个 mDNS 服务，现在你应该可以在你的 iOS 设备上的 Airplay 列表里看到一个名字为你设置的 `hostName` 的设备了。当然，现在点击连接应该是没有任何反应的，因为接下来需要进行好几次的  RTSP 请求来进行校验和连接，我们要做的也主要就是接下来这几个步骤了。

> **Airplay 连接一开始的延迟貌似是没有办法解决的**。参考这篇[SO回答](http://stackoverflow.com/questions/9997882/detecting-the-airplay-latency)，里面明确指出，Airplay 连接的延迟来源于发送方需要多次 RTSP 请求握手，大概在**两秒左右**，当然，如果你在客户端层面去做自己的传输协议当然是没有问题的，但是你并不能按照 Airplay 的包格式来实现系统级的 Airplay 到其他不论是原生的 iOS 设备还是支持了 Airplay 的 Android 设备上去，这会被苹果 Reject。所以如果在之后的开发中最后遇到了一点几秒的延迟没法解决的时候，记住不要钻进坑里了。实际测试中延迟大概在 1.6 秒左右。

## 开启你的服务端

告知了 iOS 设备你的端口信息之后，接下来就是在指定的端口开启你的服务端等候 iOS 设备传来的包了。在这里我们使用的是 Netty 库的 bootstrap 来搭建一个服务器，关于 Netty 你可以在百度和谷歌上找到更多介绍。总之它的机制是每一个新的 TCP 连接都会建立一个子的 channel 然后每一个 channel 的处理都是一个 pipeline 的处理模式，接收到消息的时候消息会在 pipeline 中流动，直到不再往下流动，发送消息反之亦然。

苹果的 Airplay 协议主要是通过 RTSP 协议的 Header 中的几个参数来进行身份的验证和包的校验，所以为了满足苹果自己需要的校验规则，我们需要在 pipeline 中加上这几个处理校验的 Handler：

```
pipeline.addLast("challengeResponse", new RaopRtspChallengeResponseHandler(NetworkUtils.getInstance().getHardwareAddress()));
pipeline.addLast("header", new RaopRtspHeaderHandler());
pipeline.addLast("options", new RaopRtspOptionsHandler());
```

其中：

- 由 iOS 设备向 Android 设备发送的 Request 的 Header 中 （注意这里你的 Android 是作为服务端的）包含一个叫做 `Apple-Challenge` 的字段，它的值需要经过 Base64 解密之后获得一个凭证，这个凭证是要在每一次的 Response 中使用到的。
- 由 Android 设备向 iOS 设备发送的 Response 的 Header 中需要包含一个叫做 `Apple-Response` 的字段，它的值需要经过一层 RSA 加密和一层 Base64 加密，原始数据则是 16 位 `Apple-Challenge` 解密后的凭证 + 16位 InetAddress.getAddress() 获取到的 byte 数组 + 6 位 硬件地址。分别是从 Request 中，`InetAddress.getAddress()` 和下面这段代码中的 `NetworkInterface` 来获得硬件地址。带有 `Apple-Challenge` Header 的包只会在 RTSP 连接建立的时候发送一次，因此稍微判断一下是否需要返回 `Apple-Response` 的 Header 就可以了。另外，在这里的 RSA 加密中用到的秘钥是一个私钥，也就是双方提前约定好的一个串，这个串会不定期的更新，破解的事情应该只有少数大神才做的了吧……对于我们主要还是从国外的一些博客和网站上经常去关注是否有私钥更新比较靠谱。这个私钥在所有的 RSA 解密操作中都要用到。

```
for(final NetworkInterface iface: Collections.list(NetworkInterface.getNetworkInterfaces())) {
if (iface.isLoopback()){
continue;
}
if (iface.isPointToPoint()){
continue;
}

try {
final byte[] ifaceMacAddress = iface.getHardwareAddress();
if ((ifaceMacAddress != null) && (ifaceMacAddress.length == 6) && !isBlockedHardwareAddress(ifaceMacAddress)) {
return Arrays.copyOfRange(ifaceMacAddress, 0, 6);
}
}
catch (final Throwable e) {
/* Ignore */
}
}
```

- 对于 RTSP Header 的处理，每个 RTSP 包都会带有 `CSeq` 的头，这个头需要在 Response 和 Request 中保持一致。它指定了 RTSP 请求回应对的序列号，在每个请求或回应中都必须包括这个头字段。对每个包含一个给定序列号的请求消息，都会有一个相同序列号的回应消息。
- 每个 RTSP Header 还要带上一个值为 `connected; type=analog` 的头 `Audio-Jack-Status`。
- 你还要响应 RTSP 的 OPTION 请求，这个请求是由客户端向服务端发起，要求服务端告知支持的所有请求类型，因此这里我们需要将所有的 RTSP 请求方法带在 Response 中返回给客户端。

## 接收并处理你的数据流

当请求经过了上面几层 Handler 还在往下传递的时候，这个时候数据包应该就到了 RTSP 的正常处理流程中了。而这些所有的关于 RTSP 的处理都是在 `AudioHandler` 中来完成的。我们会收到下面这几种请求

- ANNOUNCE 初始化步骤，传输媒体信息，编码和加密秘钥
- SETUP 连接步骤
- RECORD 不需要做什么，在这里所有的工作都在前两步里面完成了
- FLUSH 当客户端终止了 Airtunes 传输的时候发送，用来清空数据队列
- TEARDOWN 直接关闭连接

### ANNOUNCE

ANNOUNCE 中主要是带来了一些 RTP 数据的参数，Android 可以根据这些参数来初始化相应的 **RTP 处理队列**，**ALAC Decoder** 和 **AES 解密处理器**（注意所有之后的 RTP 包都是 AES 加密过的，需要用这里初始化的解密处理器解一遍，但是 RTSP 包不是 ）。ANNOUNCE 在传输的时候遵循 **SDP 描述格式**来传输媒体信息：

> **关于 SDP**
>
> SDP 是一种会话描述格式，它不属于传输协议。
>
> SDP协议是基于文本的协议，这样就能保证协议的可扩展性比较强。SDP 不支持会话内容或媒体编码的协商，所以在流媒体中只用来描述媒体信息。
>
> SDP描述由许多文本行组成，文本行的格式为:
>
> **类型 = 值**
>
> 其中，类型是一个字母，值是结构化的文本串，其格式依类型而定。
>
> **sdp的格式:**
>
> ```
> v=<version>
> o=<username> <session id> <version> <network type> <address type> <address>
> s=<session name>
> i=<session description>
> u=<URI>
> e=<email address>
> p=<phone number>
> c=<network type> <address type> <connection address>
> b=<modifier>:<bandwidth-value>
> t=<start time> <stop time>
> r=<repeat interval> <active duration> <list of offsets from start-time>
> z=<adjustment time> <offset> <adjustment time> <offset> ....
> k=<method>
> k=<method>:<encryption key>
> a=<attribute>
> a=<attribute>:<value>
> m=<media> <port> <transport> <fmt list>
>
> v = (协议版本)
> o = (所有者/创建者和会话标识符)
> s = (会话名称)
> i = * (会话信息)
> u = * (URI 描述)
> e = * (Email 地址)
> p = * (电话号码)
> c = * (连接信息)
> b = * (带宽信息)
> z = * (时间区域调整)
> k = * (加密密钥)
> a = * (0 个或多个会话属性行)
>
> 时间描述: 
> t = (会话活动时间)
> r = * (0或多次重复次数)
>
> 媒体描述: 
> m = (媒体名称和传输地址)
> i = * (媒体标题)
> c = * (连接信息 — 如果包含在会话层则该字段可选)
> b = * (带宽信息)
> k = * (加密密钥)
> a = * (0 个或多个媒体属性行)
> ```

Airplay 服务所定义的 ANNOUNCE 包的 SDP 格式如下：

```
/**
* Sample sdp content:
* 
v=0
o=iTunes 3413821438 0 IN IP4 fe80::217:f2ff:fe0f:e0f6
s=iTunes
c=IN IP4 fe80::5a55:caff:fe1a:e187
t=0 0
m=audio 0 RTP/AVP 96
a=rtpmap:96 AppleLossless
a=fmtp:96 352 0 16 40 10 14 2 255 0 0 44100
a=fpaeskey:RlBMWQECAQAAAAA8AAAAAPFOnNe+zWb5/n4L5KZkE2AAAAAQlDx69reTdwHF9LaNmhiRURTAbcL4brYAceAkZ49YirXm62N4
a=aesiv:5b+YZi9Ikb845BmNhaVo+Q
*/
```

根据样例格式我们可以解析出 AES 解密的**秘钥**和**初始化矩阵IV**以及流的数据格式，从而初始化 **ALAC Decoder**。其中，参数 m 的最后一个值和 rtpmap 的第一个值需要保持一致，rtpmap 的第一个值和 fmtp 的第一个值需要保持一致，他们都是 **payload type** 的值，因此在解析完包的数据之后要进行校验。fmtp 第一个参数之后的所有参数表示的都是媒体格式的指定参数。我们用这些参数来初始化 ALAC Decoder。*关于 SDP 的详细参数描述你可以在谷歌上找到更多*。

> a=fmtp:<format> <format specific parameters>
> ​       This attribute allows parameters that are specific to a particular format to be conveyed in a way that SDP doesn't have to understand them.  The format must be one of the formats specified for the media.  Format-specific parameters may be any set of parameters required to be conveyed by SDP and given unchanged to the media tool that will use this format.
>
> ​       It is a media attribute, and is not dependent on charset.

接下来是 AES 解密的秘钥和初始化矩阵 IV：

```
if ("rsaaeskey".equals(key)) {
/* Sets the AES key required to decrypt the audio data. The key is
* encrypted wih the AirTunes private key
*/
byte[] aesKeyRaw;

rsaPkCS1OaepCipher.init(Cipher.DECRYPT_MODE, AirTunesCryptography.PrivateKey);
aesKeyRaw = rsaPkCS1OaepCipher.doFinal(Base64.decodeUnpadded(value));

aesKey = new SecretKeySpec(aesKeyRaw, "AES");
}
else if ("aesiv".equals(key)) {
/* Sets the AES initialization vector */
aesIv = new IvParameterSpec(Base64.decodeUnpadded(value));
}
```

这两个值都是用 Base64 加密过的，所以我们要先 Base64 解密得到原始数据，然后 **AES Key** 需要再通过 Airtunes 的秘钥来 RSA 解密，最后得到 AES 解密需要的 Key。

### SETUP

在 ANNOUNCE 中我们主要是得到了数据格式，数据解密的方法参数这些基本信息，那么 SETUP 的时候客户端就是在和我们交换一些连接信息：主要也就是三个 port 的信息，对应三个 channel，分别是 **control port -> control channel**，**timing port -> timing channel** 和 **server port -> audio channel**，这是三个 **UDP 连接**的端口。这也是整个 Airtunes 服务结构中最重要的部分了：

- **control port** 是用来发送 resendTransmitRequest 的 channel，也就是当 Android 这边发现我收到的音乐流数据包中有丢失帧的时候，可以通过 control port 发送 resendTransmit 的 request 给 iOS 设备，设备收到后会将帧在 response 中补发回来
- **timing port** 用来传输 Airplay 的时间同步包，同时也可以主动向 iOS 设备请求当前的时间戳来校准流的时间戳
- **server port** 则是用来传输最主要的音乐流数据包

> 在这里我们将 control 和 timing 的包统一 reroute 到 audio 的 channel 上来处理。接收到的 UpStream 将包从 control 和 timing 集中到 audio 来处理，而发送出去的 DownStream 则是将指定类型的包从 audio 分发到 control 和 timing 去发送和接收 response。下面会详细展开。

```
/* Split Transport header into individual options and prepare response options list */
final Deque<String> requestOptions = new java.util.LinkedList<String>(Arrays.asList(req.getHeader(HEADER_TRANSPORT).split(";")));
final List<String> responseOptions = new java.util.LinkedList<String>();

/* Transport header. Protocol must be RTP/AVP/UDP */
final String requestProtocol = requestOptions.removeFirst();
if ( ! "RTP/AVP/UDP".equals(requestProtocol)){
throw new ProtocolException("Transport protocol must be RTP/AVP/UDP, but was " + requestProtocol);
}

responseOptions.add(requestProtocol);
```

> HEADER 中 key 为 **Transport** 的字段值必须为 `RTP/AVP/UDP` 。

首先对 SETUP 的参数列表进行解析，解出来的 `requestOptions` 仍然是用正则匹配的形式获取到 key - value 对：

```
/* Parse incoming transport options and build response options */
for(final String requestOption: requestOptions) {
/* Split option into key and value */
final Matcher transportOption = PATTERN_TRANSPORT_OPTION.matcher(requestOption);
if ( ! transportOption.matches() ){
throw new ProtocolException("Cannot parse Transport option " + requestOption);
}
final String key = transportOption.group(1);
final String value = transportOption.group(3);
```

其中我们只要对指定几个 key 进行 response 就可以了，其中，除了 `interleaved` 和 `mode` 返回的参数是固定的之外，`control_port` 和 `timing_port` 在 request 中所对应的 value 是客户端的端口，而 response 中需要带上服务端的端口。同时，这两个 UDP 连接由服务端发起去连接客户端对应的端口。最后再告知客户端 `server_port` 的端口。

**interleaved** 指的是由于这条 TCP 连接 RTP 和 RTCP 都要使用，因此两个连接的数据包会交叉传输在同一个 TCP 连接上，每个包都会再加一层标识，而标识 Channel 的值就由这里的 interleaved 后面的值 0-1 来决定，表示有 0 和 1 两种交叉混用的 Channel 类型。

```
/* Probably means that two channels are interleaved in the stream. Included in the response options */
if ( ! "0-1".equals(value)){
throw new ProtocolException("Unsupported Transport option, interleaved must be 0-1 but was " + value);
}
responseOptions.add("interleaved=0-1");
```

**mode** 则是校验客户端要求我们做的事情，这是 RTSP 协议中规定的一部分，在 Airplay 中，Server 永远承担的是接收数据的工作，因此 mode 的值也应当保持为 **record** 。

```
/* Means the we're supposed to receive audio data, not send it. Included in the response options */
if ( ! "record".equals(value)){
throw new ProtocolException("Unsupported Transport option, mode must be record but was " + value);
}
responseOptions.add("mode=record");
```

**control_port** 是 control channel 对应的客户端的端口号，而我们返回的 response 中需要改成服务端的端口号。可以随便分配一个比较大的端口号就行。

```
/* Port number of the client's control socket. Response includes port number of *our* control port */
final int clientControlPort = Integer.valueOf(value);

controlChannel = createRtpChannel(
substitutePort((InetSocketAddress)ctx.getChannel().getLocalAddress(), 53670),
substitutePort((InetSocketAddress)ctx.getChannel().getRemoteAddress(), clientControlPort),
RaopRtpChannelType.Control
);
responseOptions.add("control_port=" + ((InetSocketAddress)controlChannel.getLocalAddress()).getPort());
```

**timing_port** 则是 timing channel 对应的客户端的端口号。

```
/* Port number of the client's timing socket. Response includes port number of *our* timing port */
final int clientTimingPort = Integer.valueOf(value);

timingChannel = createRtpChannel(
substitutePort((InetSocketAddress)ctx.getChannel().getLocalAddress(), 53669),
substitutePort((InetSocketAddress)ctx.getChannel().getRemoteAddress(), clientTimingPort),
RaopRtpChannelType.Timing
);

responseOptions.add("timing_port=" + ((InetSocketAddress)timingChannel.getLocalAddress()).getPort());
```

**server_port** 这个 key 并不在 SETUP 的参数列表中，但是你需要在 response 中带上，告知客户端你在哪个端口打开了你的 audio 数据接收。因此它不需要主动去连接客户端的端口。

```
/* Create audio socket and include it's port in our response */
audioChannel = createRtpChannel(
substitutePort((InetSocketAddress)ctx.getChannel().getLocalAddress(), 53671),
null,
RaopRtpChannelType.Audio
);

responseOptions.add("server_port=" + ((InetSocketAddress)audioChannel.getLocalAddress()).getPort());
```

其中的 `createRtpChannel` 方法中，我们同样也为每一个端口新建一个 bootstrap 实例，添加 pipeline Handler，然后将 timing 和 control 两个 port 连接到 SETUP 包带来的 iOS 客户端端口上去。连接成功后 SETUP 也就处理完毕了。

```
/* Set pipeline factory for the RTP channel */
bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
@Override
public ChannelPipeline getPipeline() throws Exception {
final ChannelPipeline pipeline = Channels.pipeline();

final AirPlayServer airPlayServer = AirPlayServer.getIstance();

pipeline.addLast("executionHandler", airPlayServer.getChannelExecutionHandler());
pipeline.addLast("exceptionLogger", exceptionLoggingHandler);
pipeline.addLast("decoder", decodeHandler);
pipeline.addLast("encoder", encodeHandler);

/* We pretend that all communication takes place on the audio channel,
* and simply re-route packets from and to the control and timing channels
*/
if ( ! channelType.equals(RaopRtpChannelType.Audio)) {
pipeline.addLast("inputToAudioRouter", inputToAudioRouterDownstreamHandler);

/* Must come *after* the router, otherwise incoming packets are logged twice */
pipeline.addLast("packetLogger", packetLoggingHandler);
}
else {
/* Must come *before* the router, otherwise outgoing packets are logged twice */
pipeline.addLast("packetLogger", packetLoggingHandler);
pipeline.addLast("audioToOutputRouter", audioToOutputRouterUpstreamHandler);
pipeline.addLast("timing", timingHandler);
pipeline.addLast("resendRequester", resendRequestHandler);

if (decryptionHandler != null){
pipeline.addLast("decrypt", decryptionHandler);
}

if (audioDecodeHandler != null){
pipeline.addLast("audioDecode", audioDecodeHandler);
}

pipeline.addLast("enqueue", audioEnqueueHandler);
}

return pipeline;
}
});
```

这里的 pipeline 模型如图，也是三个 channel 处理流程的结构图，接下来的小节会展开说明：

![](https://ooo.0o0.ooo/2016/08/28/57c300b59b89e.png)

## Audio Pipeline 和 三个 Channel 之间的关系

SETUP 结束之后就会开始收到 Audio 的数据包了。那么正式的处理就要开始了。

根据上面这张我总结出来的流程图，Airplay Service 可以根据 bootstrap 的 pipeline 的特性可以分为 `Up Stream` 和 `Down Stream`，一个是从客户端向服务端传递的消息，一个是从服务端向客户端传递的消息。

##### Up Stream

首先不论是 Up 还是 Down Stream，都要先经过一个 Executor Handler，这个 Handler 中包括了一个线程池 Executor，当收到新的 UpStream 的数据包的时候，都会交给这个线程池来分配线程处理，在这里声明的线程池是一个 `OrderedMemoryAwareThreadPoolExecutor`。至于为什么在 Netty 的 pipeline 处理中要用到线程池来分配任务，可以参考[这篇文章](http://www.techv5.com/topic/85/)。简要地说就是由于 Handler 处理的工作量很大，为了不堵塞线程，Netty 会开好几个线程来处理，并且 `OrderedMemoryAwareThreadPoolExecutor` 能够保证处理的事件流的顺序，所以这里要加这一层。

数据进入 pipeline 之后，先是按照 RTP Packet 的格式进行 decode。在 Airplay 协议中，总共有如下几种 Packet Type：

- TimingRequest
- TimingResponse
- Sync
- RetransmitRequest
- AudioRetransmit
- AudioTransmit

其中 `TimingRequest`，`TimingResponse` 和 `Sync` 三种包类型都是属于 timing channel的，`RetransmitRequest` 是由 control channel 发起的对丢失包重传的请求，而 `AudioRetransmit` 和 `AudioTransmit` 都是由 audio channel 处理的包含了音乐数据的包。

消息继续往下传递，过了 Logger 之后就到了 router。router 维护了 audio channel 和另外两个 channel 之间的关系：router 将另外两个 channel 应该处理的包发送给对应的 Handler 去处理。

timing channel 不仅处理 Sync 数据包，同时在 channel 启动的时候也会启动一个单独的线程，每三秒钟执行一次 timing request，来确认本地时钟和客户端时钟的同步。而 control channel 做的事情则是在**每收到一个**新的 audio 数据包的时候都会**确认一次数据包的 sequence number 是否和当前的是连续的**，如果不是连续的，则将中间缺失的 number 标记为 missing 的数据包，并且向客户端发送一个 resend 的请求。当客户端发来了 `AudioRetransmit` 类型的数据包后，它的内容其实也是由 audio channel 接收的，control channcel 只是负责将刚才标记为 missing 的 sequence number 清除掉。

这两个 channel 在发送 request 的时候，也会发回到 audio channel 的 Handler 上来，通过 audio channel 这边的 encode 之后再发送出去。

而音乐数据包，则需要经过 AES 解密，这个解密器我们已经在 ANNOUNCE 的时候初始化好了，再经过 ALACDecoder，也是在 ANNOUNCE 的时候根据获得的媒体信息初始化的音频解码器，最后在 EnqueueHandler 中决定是否进入音频输出队列。

##### Down Stream

往客户端发送的信息主要就是 timine 和 control 两个 channel 发起的一些请求了，audio channel 没有参与 down stream 的传递。

## EnqueueHandler 音乐数据队列 Handler

当一个数据包经过层层解密和解析进入队列 Handler 之后，还要进行一大堆的时间戳合法性校验。**每一个数据包都包含了很多帧，每一个帧都包含了一个帧序号，而每一个包也都有一个开始的帧序号。**这里涉及到好几个地方的时间和与时间相对应的帧序列：

- Android 上  Audio Track 当前的 time
- 服务端队列中当前的 frame time
- audio channel 中客户端传来的数据包中的 frame time
- timing channel 中客户端传来的 Sync 和 timing response 包中的 frame time

首先，我们允许一定范围的延迟，因为数据的传输，最开始的握手包括 iOS 端 Airplay 的机制都可能导致一定的延迟，因此 **timing channel 最重要的作用之一就是维护和当前主队列最新 frame time的 offset**。在每一个 timing response 的包中，我可以知道当前客户端的帧序号和服务端已经播放到的帧序号的 offset，在每解析一个数据包的时候，都要使用当前的 offset 来将客户端的帧序号转换成服务端的帧序号。每个包所带的 frame time 都可能有下面三种情况：

- **太迟了**


![](https://ooo.0o0.ooo/2016/08/29/57c42046195f6.png)

太迟了的情况如图所示，Line Time 这条轴就是 Audio Track ，**Now Time** 指的是当前音乐数据已经播放到什么时间了，这时候服务端接收到的一个 Packet，在包中的开始帧序列为 frame time，将这个帧序列转换为本地 Audio Track 对应的 Line Time，加上整个 Packet 包含的帧数，这个值与 Now Time 的差距**转化成时间**就是它的 Delay，当这个 delay 大于一个包长度的时候，由于它已经是播放过的时间线了，因此当一个包迟到了一个包的长度以上，它就不再被需要了，这时候这个包也就不会被加进 queue 了。

- **太早**

![](https://ooo.0o0.ooo/2016/08/29/57c42030293f6.png)

太早的情况如图所示。有别于迟到的包，提前来的包其实是件好事，但是提前来的太早的包也不一定是件好事，它很可能要么是个错误的包，要么是个传输错误的包。因此对于提前到来的包一般都有一个时间长度的阈值，提前了大于多少秒到来的包才会被认定是 too early 然后被丢掉。在这个项目中这个阈值为 10 秒。

- **正正好，还行**

![](https://ooo.0o0.ooo/2016/08/29/57c42013d2672.png)

正正好的情况如图所示。当一个数据包上述两种情况都不满足的时候，那就说明这个包的时间戳和帧序列是我们可以接受的，于是我们将这个包加进最后的音频处理队列。

进入了队列之后，我们还要将数据从这个队列中按顺序写到 Audio Track 上去。首先确保 Audio Track 的模式为 **MODE_STREAM** ，然后按照先进先出的顺序处理队列。其实在这里的处理中对于每一个包的开始帧的序列号又做了一次校验，类似上面对于 Delay 的做法，在这里又会从队列中再次筛选掉一些无用帧。接下去在将帧最后添加到 Audio Track 上去的时候，由于我们添加到队列中的包只是上面的**正正好，还行**的情况，很有可能会出现下一个包的开始和当前 Audio Track 的 Line Time 无法完全吻合的情况出现，这时候我们就需要在缺少帧的地方补上空帧，在多余帧的地方等待一会直到帧的序号完全对上，然后再将帧写入 Audio Track，这样能够保证最终播放的帧一定是序列正确的。

## 更多

其实这篇博客没有涉及太多安卓相关的东西，说白了也只是 Airplay Server 的一种 Java 实现。在实现的基础上，掌握了 Airplay 实现的原理的话，在不论是什么平台上都可以按照相应的原理来实现一个 Airplay Server。用这个将你闲置废弃的安卓手机变成你的无线音箱吧！
