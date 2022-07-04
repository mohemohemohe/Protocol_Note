# 使用http proxy的背景

1、私网环境下，借助代理服务器上公网。

2、突破IP访问限制，例如，访问国外站点，当限制了IP时，可以使用代理访问。

3、提供访问效率，代理服务器会设置缓存，这样就节省了再次访问的成本。

4、隐藏真实IP，通过代理服务器隐藏自己机器的IP地址。

# http proxy的原理

  http代理服务器会自动提取请求数据包的HTTP Request数据，并且把HTTP Response的数据转发给发送请求的客户端

![](.\images\http proxy\http proxy原理.png)

# http proxy的分类

1、全匿名代理，不改变客户端的request fields（请求信息），使服务器端看来就像有个真正的客户浏览器在访问。客户端的真实IP是隐藏起来的。 

2、普通匿名代理，能隐藏客户端的真实IP，但会更改客户端的request fields（请求信息），服务器端有可能会被认为使用了代理。 

3、透明代理（简单代理），改变客户端的request fields（请求信息），并会传送真实IP地址。

# http proxy的协议格式

1、不使用代理
2、使用代理
3、代理ssl/tls



**HTTP/1.1协议中共定义了8种方法来以不同方式操作指定的资源：**

1、OPTIONS方法：这个方法可使服务器传回该资源所支持的所有HTTP请求方法。用'*'来代替资源名称，向Web服务器发送OPTIONS请求，可以测试服务器功能是否正常运作。

2、HEAD方法：与GET方法一样，都是向服务器发出指定资源的请求。只不过服务器将不传回资源的本文部份。它的好处在于，使用这个方法可以在不必传输全部内容的情况下，就可以获取其中“关于该资源的信息”(元信息或称元数据)。

3、GET方法：向指定的资源发出“显示”请求。使用GET方法应该只用在读取数据，而不应当被用于产生“副作用”的操作中，例如在Web Application中。其中一个原因是GET可能会被网络蜘蛛等随意访问。参见安全方法

4、 POST方法：向指定资源提交数据，请求服务器进行处理(例如提交表单或者上传文件)。数据被包含在请求本文中。这个请求可能会创建新的资源或修改现有资源，或二者皆有。

5、PUT方法：向指定资源位置上传其最新内容。

6、DELETE方法：请求服务器删除Request-URI所标识的资源。

7、TRACE方法：回显服务器收到的请求，主要用于测试或诊断。

8、CONNECT方法：HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器。通常用于SSL加密服务器的链接(经由非加密的HTTP代理服务器)。

connect方法的通常就是把服务器作为跳板机，让服务器直接代理客户端访问，然后把数据原原本本的返回给客户端，connect方法的原理就是TCP直连。



**具体的格式如下：**

## 1、不使用代理

```
GET / HTTP/1.1
Accept: application/x-ms-application, image/jpeg, application/xaml+xml, image/gif, image/pjpeg, application/x-ms-xbap, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*
Accept-Language: zh-CN
User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; WOW64; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; InfoPath.2; .NET4.0C; .NET4.0E)
Accept-Encoding: gzip, deflate
Host: 192.168.1.10
Connection: Keep-Alive
Cookie: CNZZDATA1252995935=579722882-1456319452-%7C1472176395;
```



## 2、使用代理

```
GET http://192.168.1.10/ HTTP/1.1
Accept: application/x-ms-application, image/jpeg, application/xaml+xml, image/gif, image/pjpeg, application/x-ms-xbap, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*
Accept-Language: zh-CN
User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; WOW64; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; InfoPath.2; .NET4.0C; .NET4.0E)
Accept-Encoding: gzip, deflate
Proxy-Connection: Keep-Alive
Host: 192.168.1.10
Cookie: CNZZDATA1252995935=579722882-1456319452-%7C1472176395;
```

Accept：请求报头域用于指定客户端接受哪些类型的信息。

Accept：text/html，表明客户端希望接受html文本。

Accept-Language：请求报头域类似于Accept，但是它是用于指定一种自然语言。

User-Agent：请求报头域允许客户端将它的操作系统、浏览器和其它属性告诉服务器。如果不使用User-Agent请求报头域，服务器端就无法得知客户端的属性信息。

相比不适用代理的情况下，协议头多了`Proxy-Connection: Keep-Alive`。

HTTP协议中Connection是用来对HTTP协议连接进行的说明，如果有多个说明，使用英文逗号分隔，Proxy代表了代理。



## 3、代理ssl/tls

```
CONNECT http://192.168.1.10:80/ HTTP/1.1
Accept: application/x-ms-application, image/jpeg, application/xaml+xml, image/gif, image/pjpeg, application/x-ms-xbap, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*
Accept-Language: zh-CN
User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; WOW64; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; InfoPath.2; .NET4.0C; .NET4.0E)
Accept-Encoding: gzip, deflate
Proxy-Connection: Keep-Alive
Host: 192.168.1.10
```

 connect是在HTTP1.1协议上才新增的方法，用于支撑https加密。 

# http proxy抓包

工具：
  客户端：浏览器，设置代理
  服务器端：burpsuite设置代理服务器端