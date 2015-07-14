---
layout: post
title: "Gson源码分析(一)之Json结构抽象和注解使用"
date: 2015-01-06 22:31:27 +0800
comments: true
categories: Android
---
XML和Json作为最常用的两种网络传输格式而被广泛使用，XML在早期数据传输中作为首选，但是近年来Json以其轻量级和更容易编写和解析而越来越流行，Gson作为google的一个开源Json解析框架提供了稳定和快速解析的功能，可以读读它的源代码了解一番。
<!--more-->

说到Gson，其实它无非就是做两个工作，序列化(Object--->JsonString)和反序列化(JsonString--->Object)，后文所说的*两个方向*从Object到String和String到Object的两个方向。可想而知，对于序列化来说，是较为容易的工作，而对于反序列化即Json解析才是Gson的重头戏。既然是对Json字符串的解析，那么少不了对Json字符串中的结构进行抽象。

##Json抽象类

###JsonElement

这是Json中元素的基类，它只提供了若干个类型判断的接口，简单判断这个Json元素的类型。以下几个类型都是它的子类。

####1、JsonObject

包含多个JsonElement的集合，它在Json中对应这种类型的数据：

	{
		"count":100,
		"users":[],
		"paging":{
			"offset":0,
			"limit":10,
			"hasMore":true
		}
	}
	
这个data是一个典型的JsonObject，它以`{`开头，其中包含了一些类似数值，数组，对象等其他JsonElement的内容。**其实每一个Json字符串的根节点都是一个JsonObject或者JsonArray。**

JsonObject提供了比较多的方法来得到Json中的信息，addProperty()函数可以在当前Json节点下新建子结点。

#### 2、JsonArray

JsonArray也表示JsonElement的集合，注意：**Json中的数组并不要求所有的数据类型都一样**。

		[true,"hello"] //JsonArray,包括一个boolean和一个hello类型。
		
需要讨论的是JsonArray和JsonObject的区别是什么？从集合的角度来说，JsonObject中的JsonElement是无序的，而JsonArray中的集合元素是有序的，从直观感受来说，你可以通过下标来引用JsonArray中的元素，而JsonObject是通过键值对的方式来访问的,`get("name")--->value`。

#### 3、JsonPrimitive

对应Json中的基本类型，比如boolean，int，当然提供了基本类型和包装类的自动替换。

	"count":100

####4、JsonNull

空，对应null

	"person":null

以上就是Gson对应Json结构的封装。

###注解-Annotations

####Expose

在对象进行序列化和反序列化的过程中，我们可以通过注解来屏蔽某一些字段。这个注解默认有两个参数，`serialize`和`deserialize`都是默认true。如果设置为false，表示这个序列化(反)的过程中，这一个属性不需要进行处理。

通过Expose标注的属性在直接`new Gson()`的情况下不能生效，我们必须通过`Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation().create()`来创建一个可以识别Expose注解的Gson。

小小的吐槽一把，这个地方的使用确实不太方便，举个栗子，一般来说，我们进行序列化的时候都是希望某一个属性不会序列化到Json字符里面(反之亦然)，所以这里的一般思维是我要去处理这些特殊的属性。而如果你想通过Expose来去掉10个属性中的某一个，对不起，10个属性你都需要加上`@Expose`,然后对你想要处理的那个属性的Expose注解增加false参数，简直就是坑爹。。。。

	//我想让Person类在序列化时，不去序列化password，是不是很坑爹？
	class Person {
		@Expose
		private String userName;
		@Expose (serialize = false)
		private String passWord;
	}

此外，和java本身的序列化一样，如果一个属性设置为`transient`或者`static`,那么两个序列化的两个方向上都会屏蔽掉这个属性，虽然比Expose简单，但是不够灵活。

####SerializedName
这个注解使用较多，它会改变两个方向上这个属性的名称，在序列化是，JsonElement的键值会被替换成这个名字；解析时，Json中键值为这个名字的JsonElement会赋值给被注解的属性。

	class Person {

    @SerializedName(value = "A")
    private int a ＝ 1;
    private int b = 2;
    }
    
    //{"A":1,"b":2}  这个Json字符串和前面的Person等价
它使用场景最多的地方就是比如后端返回的json中的名称和我们定义的model类名称不一样时使用。

####Since 和 Until

我们可以对我们的Model类进行序列化(两个方向)的版本控制，Since和Until刚好是两个相反的意义。

例子：

	class Person {
		@Since(value = 1.0)		//GsonBuilder指定版本要从1.0开始的Gson才能解析
		private int a = 1;
		
		@Until(value = 1.5)   //GsonBuilder指定版本到1.5的Gson都可以解析，超过了不能解析
		private int b = 2;
	}
	
和Expose一样，要想Gson识别这两个注解，同样需要通过GsonBuilder.setVersion(double).create()来实现。


