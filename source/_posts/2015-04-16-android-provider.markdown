---
layout: post
title: "Android Content Provider"
date: 2015-04-16 10:58:35 +0800
comments: true
categories: Android
---
`ContentProvider`作为Android应用之间共享数据的桥梁，为共享数据提供了一个统一的访问方式(一套统一的接口)。
<!--more-->

## URI
在Android中大量使用URI作为资源定位标识，它主要由 `scheme host(authority) path params组成`。例如我们有如下的一个`Content Provider`的URI如下：

	content://www.chuyun923.com/books/10086/title/firstWord
	

其中`content`，是Android为Content Provider规定的scheme；
PATH(`books/10086/title/firstWord`)可以直接表示用来操作的业务，表示书籍`_ID=10086`的书名的第一个单词。**注意，我们可以在URI中定义各种奇葩的PATH来描述业务，但是的最终解释权归Content Provider的定义**

既然我们可以指定一些PATH表示业务，那么`Content Provider`必然需要去解析各种PATH，然后提供各种匹配的处理逻辑，此时，UriMatcher应运而生。

### UriMatcher

对一个给定的Uri进行HOST和PATH的匹配，而Scheme不在考虑范围。对于PATH的匹配主要是挨个去比较各个PATH参数是否相等。

使用UriMatcher，首先需要给它配置一些配置URI，以及映射的`int`。

	    private static final UriMatcher uriMatcher;
    	static{
        	uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        	uriMatcher.addURI("net.manoel.provider.Books", "books", 1);
        	uriMatcher.addURI("net.manoel.provider.Books", "books/#/title", 2);
        	uriMatcher.addURI("net.manoel.provider.Books","books/#/title/firstword",3);
    	}
    	
这里配置了三个业务PATH，第一个表示整个`books`整个表。第二URI表示指定Id的书名，第三个表示指定id的书名的首单词。

那么，
`books/12/title`将匹配第二个，在`Content Provider`收到这个`URI`时，可以使用`UriMatcher`得到2。
UriMatcher为URI提供了两种通配符 `#`代表数字；`*`代表任意字符。

**需要注意一点的是**，`UriMatcher`进行匹配，会受到`addURI`的先后顺序影响。

如：`uriMatcher`按以下顺序添加：
	
	*/#
	#/*
	
那么 `9/title`将不能获得匹配，是不是很奇怪？

    */#/#
    */*/*
    
 `books/10/title`也不能被匹配。
 
在匹配PATH是挨个参数进行，一旦一个路径参数被匹配，那么它就不会更换。
以`books/10/title`来说，匹配第一个patten的第一个参数 * 通过，10 匹配 `#`也通过，title不能匹配#，那么此时进行第二个patten的匹配。那么第二个patten必须满足`*/#`开头，否则就算是 `*/*`也不能通过。





