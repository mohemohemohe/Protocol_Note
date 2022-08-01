# websocket协议及原理详解

## 定义及背景

websocket协议是一种服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，实现客户端和服务器端真正的双向平等对话，是服务器推送技术的一种。

websocket协议在握手阶段是通过http协商的，可以轻松通过各种代理，不易被屏蔽；它通信的数据格式轻量、性能开销小，通信高效；另外，websocket不像http会受到同源的限制，因此websocket客户端能够与任何服务器端通信。

注：同源限制指的是5元组相同，5元组分别包括：源IP、目标IP、源端口、目的端口、协议。

## 交互过程及原理

整个交互过程分成两个阶段：1、http handshake； 2、websocket实现数据的传输。如图所示：
![image](https://user-images.githubusercontent.com/45613769/181906433-cc8554f1-eac8-418e-8ed1-c7c9bd27c335.png)

1、http handshake阶段

通过Upgrade实现了协议的升级，并且通过header头协商了一些扩展参数：

client--> server: 客户端向服务器端发起协议升级请求，请求内容为 Upgrade: websocket

server--> client: 服务器端支持websocket，就会返回101响应码；若支持对应的扩展参数，就会在响应头中携带对应的参数。

2、最后使用websocket协议进行数据的传输



## 协议格式

### 一、http handshake

http握手阶段，http协议格式就不细说了，我们重点介绍这个阶段，握手协商了哪些参数，以及这些参数将会如何作用到后面的websocket数据传输的过程中。

1、websocket协议请求：ws://(default port:80), wss://(default port=443)，这种方式类似HTTP和HTTPS协议的请求方式，但是不同的是，websocket不仅仅用于浏览器的访问。当请求头出现下面的头部信息时，就说明发起了websocket协议的请求:

```
Connection: Upgrade
Upgrade: websocket
```

2、Sec-WebSocket-Version表示websocket的版本，若服务器端不支持客户端请求的版本，服务器端就会向客户端返回自己所支持的版本号，这时客户端会用它所支持的最新版本号重新进行http握手，相关的交互字段如下：
①client->server:

```
Sec-WebSocket-Version: 13
```

②server->client: 服务器端不支持版本13，因此服务器端返回自己所支持的版本号列表

```
HTTP/1.1 400 Bad Request
Sec-WebSocket-Version: 8,7,5
```

③client->server:客户端选择自己所支持的最新的版本号7，重新进行握手

```
Sec-WebSocket-Version: 7
```

④Sec-Websocket-Extentsions头支持下面这些参数：

```
   o  " permessage-foo"	表示支持foo算法压缩扩展，目前官方只支持deflate的标准，以后也可能会支持LZ4、ZIP等
   
   o  "server_no_context_takeover" 服务器端不共享压缩状态，每次发送一个payload经过解压缩后，就释放状态
  
   o  "client_no_context_takeover" 同上，客户端不共享状态

   o  "server_max_window_bits" 服务端压缩算法的最大压缩窗口

   o  "client_max_window_bits" 客户端压缩算法的最大压缩窗口
```

⑤Sec-WebSocket-Key 是由浏览器随机生成的，提供基本的防护，防止恶意或无意的连接。

⑥Sec-WebSocket-Protocol (可选)，客户端向服务器端发送自己支持的自协议列表，服务器端选择自己所支持的子协议，http握手完成后，会在websocket发送阶段，payload是服务器端选择的子协议形式发送。
client->server:
![image](https://user-images.githubusercontent.com/45613769/182169914-23e80f6f-9164-4acd-a308-e0acdbbeecd8.png)
server->client:
![image](https://user-images.githubusercontent.com/45613769/182170083-fbf3c8fb-bbb6-42c5-9f60-0290b279ad70.png)


根据上面协商的压缩算法，在websocket传输数据的过程中压缩payload。
![image](https://user-images.githubusercontent.com/45613769/181906726-6bdf83f3-31fd-48b9-866a-d89d2ccbabbd.png)

![image](https://user-images.githubusercontent.com/45613769/181906767-425a915b-2b1e-4f63-8ebd-383a7dd4148e.png)


## 二、websocket阶段

### 2.1 协议格式概览

总体的协议格式如下：

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+

前面的几个字节：从FIN到Masking-key这些字段都是为payload data服务的，下面我们来看看各个字段道理是怎么作用在payload data上的。

**1、FIN（1bit），用于描述消息是否结束。**

FIN=1，表示接收方已经完整地收到了消息；

FIN=0，表示接收方还需要继续监听接收其余的数据帧。

**2、RSV1、RSV2、RSV3（各1bit），3个保留位，用于描述扩展功能。**

在不支持扩展的情况下，这3个扩展为都为0。这3个值的含义由扩展定义，如当支持压缩扩展时，第一位RSV1为0，如图所示：

**3、opcode(4bit)，用于表示交互过程中操作的类型，决定了接收方如何解析payload。**

opcode=0，表示一个延续帧，即本次数据传输采用了分片的方式；

opcode=1，表示是一个文本帧；

opcode=2，表示是一个二进制帧；

opcode=3-7，保留操作码，用来定义后续的非控制帧；

opcode=8，表示连接断开；

opcode=9，表示这是一个ping操作；

opcode=0xA，表示这是一个pong操作；

opcode=0xB-0xF，保留操作码，用来定义后续的非控制帧。



这里单独把opcode=0的情况单独拉出来分析：细心的宝宝可能会发现，**FIN=0** 和 **opcode=0** 的情况都表示数据传输采用了分片的方式，那么它俩到底有什么猫腻，到底谁说了算呢？

FIN=0 & opcode=1，表示发送的是文本类型，且消息还没发送完成，还有后续的数据帧；

FIN=0 & opcode=0，表示消息还没发送完成，还有后续的数据帧，当前的数据帧需要接在上一条数据帧之后；

FIN=1 & opcode=0，表示消息已经发送完成，没有后续的数据帧，当前的数据帧需要接在上一条数据帧之后，服务端可以将关联的数据帧组装完成的消息。



MDN的例子如下：

```
Client: FIN=1, opcode=0x1, msg="hello"
Server: (process complete message immediately) Hi.
Client: FIN=0, opcode=0x1, msg="and a"
Server: (listening, new message containing text started)
Client: FIN=0, opcode=0x0, msg="happy new"
Server: (listening, payload concatenated to previous message)
Client: FIN=1, opcode=0x0, msg="year!"
Server: (process complete message) Happy new year to you too!
```

总结：是否分片发送是根据FIN的值来判断的。

**4、MASK(1bit)，表示是否对payload进行掩码操作。**

MASK=1，表示需要对payload进行掩码计算，并且Masking-key为4bytes；

MASK=0，表示无需对payload进行掩码计算，并且Masking-key为0bytes。



**5、payload length+Extended payload length（不定长），是payload的长度。**

payload length 和 Extended payload length一起表示payload的长度，下面是最终的payload 长度的算法：

如果：

		payload length = 0~126，那么payload的长度就是payload length，即Extended payload length为0；
	
		payload length = 126，那么payload的长度取后2个字节；
	
		payload length = 127，那么payload的长度取后8个字节；

我们可以看到payload长度上限高达2G=2^(32-1)。



**6、Masking-key（4bytes），由客户端选择的随机数。**

首先，假设：

- original-octet-i：为原始数据的第i字节。
- transformed-octet-i：为转换后的数据的第i字节。
- j：为`i mod 4`的结果。
- masking-key-octet-j：为mask key第j字节。

算法描述为： original-octet-i 与 masking-key-octet-j 异或后，得到 transformed-octet-i。

> j = i MOD 4
> transformed-octet-i = original-octet-i XOR masking-key-octet-j

Js脚本实现MASK计算：

```javascript
/**
 * Unmasks a buffer using the given mask.
 *
 * @param {Buffer} buffer The buffer to unmask
 * @param {Buffer} mask The mask to use
 * @public
 */
function _unmask(buffer, mask) {
  // Required until https://github.com/nodejs/node/issues/9006 is resolved.
  const length = buffer.length;
  for (var i = 0; i < length; i++) {
    buffer[i] ^= mask[i & 3];
  }
}
```

**7、payload，是真正交互的数据**。

# 参考资料

websocket协议标准  https://tools.ietf.org/html/rfc6455 

压缩扩展参数 https://datatracker.ietf.org/doc/html/rfc7692#section-7.2.2

中文翻译的websocket协议RFC https://blog.csdn.net/BelugaW/article/details/117379133
