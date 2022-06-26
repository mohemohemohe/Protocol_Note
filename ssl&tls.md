# HTTPS的认证过程

双向认证，顾名思义，客户端和服务器端都需要验证对方的身份，在建立HTTPS连接的过程中，握手的流程比单向认证多了几步。单向认证的过程，客户端从服务器端下载服务器端公钥证书进行验证，然后建立安全通信通道。双向通信流程，客户端除了需要从服务器端下载服务器的公钥证书进行验证外，还需要把客户端的公钥证书上传到服务器端给服务器端进行验证，等双方都认证通过了，才开始建立安全通信通道进行数据传输。



## 1. 原理

## 1.1 单向认证流程

单向认证流程中，服务器端保存着公钥证书和私钥两个文件，整个握手过程如下：

![img](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/18611/1585034778516-6e938349-9008-4940-b24f-3ceb74f57fd6.png#alt=undefined)



1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务器端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），取出了服务端公钥；
4. 客户端生成一个随机数（密钥R），用刚才得到的服务器公钥去加密这个随机数形成密文，发送给服务端；
5. 服务端用自己的私钥（server.key）去解密这个密文，得到了密钥R
6. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。



## 1.2 双向认证流程



![img](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/18611/1585034830354-cf4e77f6-e87c-4bfd-9fb5-e72746f2dcd1.png#alt=undefined)



1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），取出了服务端公钥；
4. 客户端将客户端公钥证书（client.crt）发送给服务器端；
5. 服务器端使用根证书（root.crt）解密客户端公钥证书，拿到客户端公钥；
6. 客户端发送自己支持的加密方案给服务器端；
7. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密8. 后发送给客户端；
8. 客户端使用自己的私钥解密加密方案，生成一个随机数R，使用服务器公钥加密后传给服务器端；
9. 服务端用自己的私钥去解密这个密文，得到了密钥R
10. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。



## 2. 证书准备



从上一章内容中，我们可以总结出来，整个双向认证的流程需要六个证书文件：



- 服务器端公钥证书：server.crt

- 服务器端私钥文件：server.key

- 根证书：root.crt

- 客户端公钥证书：client.crt

- 客户端私钥文件：client.key

- 客户端集成证书（包括公钥和私钥，用于浏览器访问场景）：client.p12



所有的这些证书，我们都可以向证书机构去申请签发，一般需要收取一定的证书签发费用，此时我们需要选择大型的证书机构去购买。如果只是企业内部使用，不是给公众使用，也可以自行颁发自签名证书



# SSL/TLS协议格式

这里不会具体总结ssl/tls的协议交互过程和具体的协议格式，而是总结一些自己在理解时比较费劲的地方。

这是wireshark中的tls解码的注释，从这里我们可以看到，ssl/tls的版本号的变化及对应版本对应的rfc文档。

 * Supported protocol versions:
 *
 *  TLS 1.3, 1.2, 1.0, and SSL 3.0. SSL 2.0 is no longer supported, except for
 *  the SSL 2.0-compatible Client Hello.
 *
 * Primary protocol specifications:
 *
 *  https://tools.ietf.org/html/draft-hickman-netscape-ssl-00 - SSL 2.0
 *  https://tools.ietf.org/html/rfc6101 - SSL 3.0
 *  https://tools.ietf.org/html/rfc2246 - TLS 1.0
 *  https://tools.ietf.org/html/rfc4346 - TLS 1.1
 *  https://tools.ietf.org/html/rfc5246 - TLS 1.2
 *  https://tools.ietf.org/html/rfc8446 - TLS 1.3

## ssl/tls协议中的版本号

版本号列表如下：

```c++
#define SSLV2_VERSION           0x0002 /* not in record layer, SSL_CLIENT_SERVER from
                                          http://www-archive.mozilla.org/projects/security/pki/nss/ssl/draft02.html */
#define SSLV3_VERSION          0x300
#define TLSV1_VERSION          0x301
#define GMTLSV1_VERSION        0x101
#define TLSV1DOT1_VERSION      0x302
#define TLSV1DOT2_VERSION      0x303
#define TLSV1DOT3_VERSION      0x304
#define DTLSV1DOT0_VERSION     0xfeff
#define DTLSV1DOT0_OPENSSL_VERSION 0x100
#define DTLSV1DOT2_VERSION     0xfefd
```

tls1.3的草案不遵循上述的规律，具体规则如下：

```c++
/* Returns the TLS 1.3 draft version or 0 if not applicable. */
static inline guint8 extract_tls13_draft_version(guint32 version) {
    if ((version & 0xff00) == 0x7f00) {
        return (guint8) version;
    }
    return 0;
}
```

## ssl/tls服务器端将证书发送多个证书给客户端

![1656253065068](.\images\pcap-两个证书.png)在ssl/tls服务器端向客户端发送公钥证书时，其实是发送证书链，也就是说发送的证书中包含服务器端的证书、中间认证的证书（也可能没有）、根证书。



## 证书指纹

证书指纹是通过将证书转成der编码的二进制串，其实就是网络中传输的证书格式，然后在通过sha1(40字节)、sha256（64字节）得到的。

为了验证上面的猜想，我们做一个实验，点击这里可以看详情：https://security.stackexchange.com/questions/14330/what-is-the-actual-value-of-a-certificate-fingerprint

```
openssl x509 -in cert.crt -outform DER -out cert.cer //将证书转成der编码的文件
sha1sum -b cert.cer //得到sha1后的指纹
```



## tls1.3 0-RTT是怎么回事？

0-RTT并不是像我想象的那样，无需进行握手和认证协议的过程，而是通过sessionid，可以在协商过程中，就开始数据传输，具体看下面这个截图：

![1656253721827](.\images\pcap-tls1.3-0-rtt.png)

![1656253753551](.\images\pcap-tls1.3-0-rtt-application.png)

再附上一个证书的格式说明：http://www.infotech.vip/web-https.html