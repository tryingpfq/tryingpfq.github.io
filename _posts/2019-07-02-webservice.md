---
layout:     post
title:      WebService
subtitle:   WebService的实现原理
date:       2019-07-02
author:     tryingpfq
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - RPC
---



> weabService是一个轻量级独立通信技术，其实也是早期的一个RPC框架，主要是基于Web的服务，可以跨平台、语言的，相比于RMI这是很大的优势。

### WebService中的一些概念

**WSDL**

wsdl(web service definition language),webservice定义语言。
webservice服务需要通过WSDL文件来说明自己有什么服务是可以调用的，并且有哪些方法、方法参数，wsdl是基于XML去定义的。 

* 1：对应一个.wsdl的文件类型
* 2：定义了webservice的服务端和客户端应用进行交互的传递数据和响应数据格式和方式。
* 3：一个webservice对应唯一一个wsdl文档。

**SOAP**

soap(simple object access protoal),简单对象传输协议，webservice通过http协议发送和接收请求时，发送的内容（报文）和接收的内容都是采用xml格式进行封装，这些特定的http消息头和xml内容格式就是SOAP协议。


**SEI**

SEI（webservice endpoint interface webservice的终端接口）
webservice服务端用来处理请求的接口，也就是发布出去的接口。

