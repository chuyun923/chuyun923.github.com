---
layout: post
title: "HttpClient"
date: 2014-12-16 12:20:26 +0800
comments: true
categories: 网络协议/HTTP
---
Http协议是在开发中使用最多的网络协议，大部分的网络接口都是通过Http来设计。在Android中提供了两种Http客户端，HttpClient和HttpUrlConnetion。其中后者在低版本存在某些问题，但是HttpClient尽管功能强大，但使用麻烦，接口众多，本文主要是简单介绍HttpClient中使用的一些概念。
<!--more-->
##Request URI
URI用来表示一个HTTP请求的目标，它由protocol scheme,host名字,port(可选)，resource path,query(可选)和fragment(可选)组成。对应的例子就是：<br />
"http://www.google.com/search?hl=en&q=httpclient&btnG=Google+Search&aq=f&oq="
<br />
apache的HttpClient中(注意，这里将这个apache对于Http的客户端实现的类库统称为HttpClient),提供了URIBuilder来创建一个request URI。	
##HttpRequest
HttpRequest类对应于Http协议中的request line，它主要包括的信息有：Http方法、请求的URI和Http的版本号。
##HttpResponse
Httpresponse对应于Http协议中的返回中的第一行，包括：协议版本，状态码和状态码的解释。
##HttpMessage
对应于Http的请求头，所以HttpRequest和HttpResponse都实现了HttpMessage，因为在其中都有Http Head。
##Http entity
一个Http请求或者响应实际传输的内容，对于请求来说，HTTP规范定义了两种请求方法需要entity：post和put；HTTP期望所有的response都能包含一个entity，当然如果在Head中包含一些出错信息就另当别论。<br />
在HttpClient中依据来源不同，将entity分为三类：

+ streamed：从一个传输流中获得，一般是来自http连接。不可重复。
+ self-contained：存在内存中，独立于http连接。可以重复。
+ wrapping：从其他的HttpEntity中获得。依赖于获取的类。

注意：以上所说的重复不重复，指的是可不可以多次读取其中的内容，只有self-contained类型的可以重复读取。
###读取entity
推荐使用流的方式来操作entity中的内容,getContent或者writeTo。HttpClient提供了一个entityUtils类来直接将entity转化成String或者byte数组。不建议使用entityUtils，除非这个响应来自一个可信的server同时可以知道它的长度有限。

如果一个entity需要被多次读取，我们可以使用一个BufferedHttpEntity来将entity缓存至内存中。
###构造entity
对于post或者put这种可以在请求中附带entity的request(entity enclosing request)，HttpClient提供了多种针对不同数据类型的Entity，比如StringEntity、ByteArrayEntity、InputStreamEntity和FileEntity。

为了模拟HTML表单提交的Post请求，我们可以通过对Post请求设置一个UrlEncodedFormEntity给Post的entity。
##释放资源
在Http请求得到response之后，我们需要释放资源，可以选择释放content Stream 和 response自身。两者的区别在于：contentStream.close()会保留底层的连接，而response则会关闭整个连接。如果对一个response 返回的内容我们只需要其中一部分的内容，而不需要剩余的部分，可以通过直接关闭response。

注意，我们可以在一个HttpClient执行的request设置一个response handler，然后在handleResponse中处理数据，这样做的好处就是HttpClient会自动去维护资源的释放。
##异常处理
在HttpClient中，有两类异常：I/O Exception 和 HttpException。前者不是致命的错误，而且也可以恢复，而后者则相反。

   
