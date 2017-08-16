---
layout:     post
title:      "Wireshark - HTTPS握手现形记"
subtitle:   "SSL Handshake"
author:     "Chris"
header-img: "img/post-bg-5.jpg"
tags:
    - HTTP
---

# Wireshark - HTTPS握手现形记

## 前言

本文章使用Wireshark抓取支付宝的首页 [https://www.alipay.com](https://www.alipay.com) ，以观察SSL的握手细节。在本次分析中支付宝采用的Cipher Suite是TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256。

![](/img/in-post/wireshark-ssl/ssl-handshake-steps.png)

## Client Hello

客户端首先会发起Client Hello阶段的请求，与服务端磋商加密组件。我们用Wireshark抓取的报文中可以清楚地看到Cipher Suites就是客户端所支持的14种组件。而Random Bytes就正如其名，是客户端生成的随机数，尔后将会用于生成session key。

![](/img/in-post/wireshark-ssl/client-hello.jpg)

## Server Hello

服务端收到Client Hello消息后就马上响应一个Server Hello给客户端，主要告知客户端所选择加密组件和**服务端的随机数**，可以分别在下图Cipher Suite和Random Bytes中观察得到。

![](/img/in-post/wireshark-ssl/server-hello.jpg)

注意，到目前为止，客户端和服务端都拥有自己和对方的随机数，但都是明文传输的。

## Certificate

服务端发送自己的证书给客户端，其中包括完整的证书链，供客户端校验。在测试支付宝的主页当中，支付宝响应了3张证书，在Certificates[0].signedCertificate.issuer中可以清楚地看到他的上级是`Symantec Class 3 Secure Server CA`，通过这样的递归查询我们就可以知道一条自下往上的完整的证书链是如何构成的。下图显示了Certificate请求的报文概要。

![](/img/in-post/wireshark-ssl/certificate.jpg)

## Server Key Exchange

回忆一下，由Server Hello中协商的Cipher Suite为TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256，那么何为ECDHE呢？在维基百科上我们找到这个算法的全称**Elliptic curve Diffie–Hellman** (**ECDH**)，可看作是DH算法的一种升级，同样是需要双方交换公钥的。而Server Key Exchange这一步就是服务端发送公钥及其签名的操作（不发送签名则无法证明该公钥是所请求服务端所生成），这步操作与pre-master secret相关。需要注意的是，这个被交换出去的公钥与在Certificate报文中的证书公钥是两回事，在Server Key Exchange阶段发送给客户端的公钥是DH算法的一部分，尔后将会与客户端的另一半公钥生成一个新的共享密钥，用于计算生成pre-master secret。

![](/img/in-post/wireshark-ssl/server-key-exchange.jpg)

## Server Hello Done

最后服务端会发送一个Server Hello Done消息给客户端，表示Server Hello消息结束了。

![](/img/in-post/wireshark-ssl/server-hello-done.jpg)

## Client Key Exchange

在确认证书是可信的，并且是属于正要访问的网站之后，客户端将会发送DH算法中另一半公钥给服务端。此时，根据DH算法客户端和服务端都可以计算出pre-master secret（在[cloudflare的文章](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/)中ECDH密钥交换算法中所得的shared key就被认为是pre-master key）。用pre-master key还有客户端和服务端的两个随机数就可以计算出session key。

![](/img/in-post/wireshark-ssl/client-key-exchange.jpg)

## Change Cipher Spec

告知对方自己已经准备完毕，可以用磋商后的session key加密报文并进行传输了。因为属于告知事件，所以整个报文的长度就只要1个字节就可以了。

![](/img/in-post/wireshark-ssl/change-cipher-spec.jpg)

## 总结

下图是cloudflare对基于Diffie Hellman算法的SSL握手的生动总结，pre-master secret和session key旁边的注释是需要仔细研读的。

![](/img/in-post/wireshark-ssl/ssl-handshake-diffie-hellman.jpg)

至此关于Wireshark分析SSL握手报文就全部结束。