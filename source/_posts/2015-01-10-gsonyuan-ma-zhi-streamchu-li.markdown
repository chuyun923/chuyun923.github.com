---
layout: post
title: "Gson源码(二)之JsonReader和JsonWriter"
date: 2015-01-10 14:17:21 +0800
comments: true
categories: Android
---
前文介绍了`Gson`对于`Json`规范中的类型进行了抽象定义，本文将来介绍`stream`包中的源码，这一块是`Gson`的核心代码之一。Gson中的JsonReader和JsonWriter抄自android.util包中的两个类，只是把其中的上下文栈由ArrayList换成了数组，提高了效率。这两个类为Json序列化和反序列化提供了基本接口。

<!--more-->
#反序列化---JsonReader

##StringPool--(Flyweight Pattern)
在Stream包中提供了一种针对String的优化处理，可以减少内存中String对象的个数，途径就是通过StringPool来对字符串进行复用。需要注意的是，在Java中字符串对象是不能被修改的，这也是这个Pool能正常工作的重要原因。StringPool原理和HashMap一模一样，通过一个2的n次方(512)作为数组的长度，字符串的hascode计算方式和String中的一致。为了使得hashcode分布分散一点，同样使用hashmap的补偿hash函数对hashcode进行处理，这一块的具体优化处理可以关注以后的HashMap源码分析。

##深度优先的解析策略

Gson解析Json字符采用的是深度优先策略，下面是一个简单解析的栗子：

	[
	<!--数组中每一个对象为一个message对象-->
    {
      "id": 912345678901,
      "text": "How do I read a JSON stream in Java?",
      <!--geo是一个double类型的数组-->
      "geo": null,
      <!--user对象-->
      "user": {
        "name": "json_newb",
        "followers_count": 41
       }
    },
    {
      "id": 912345678902,
      "text": "@json_newb just use JsonReader!",
      "geo": [50.454722, -104.606667],
      "user": {
        "name": "jesse",
        "followers_count": 2
      }
    }
    ]
    
深度解析的逻辑如下，先找到这个json字符串的开始位置，本例中是`[`，此时我们知道这个json字符串是一个数组，下一步从第一个元素开始解析(一层循环)。读取下一个字符，`{`,可以知道第一个元素为对象。对于`{`开始的JsonObject，我们知道它其中的内容肯定是键值对的样式，那我可以读取下一个字符，而且它必须为`"`(属性名)，然后去读取属性值。ok，我们可以看到这是一种非常正常的解析逻辑，其实Gson的解析就是这么做的。

##数据抽象

为了进行上面的解析过程，Gson中定义了两种数据类型：

###JsonToken(解析类型)
由于Json定义规范的原因，我们在解析过程中只要解析到一个结构的第一个元素再联系上下文就可以推断这个结构的类型。JsonToken用来描述`Json`中的`'{','}','[',']','"'`,`NULL`,`number`，`boolean`,属性名和属性字符串值，其实JsonToken的基本上等于每一个结构的开头，其中`"`既可以表示一个属性名也可以表示一个属性就是一个String，这需要按照上下文来确定。
在`JsonReader`定义了一个属性`token`，用来存储`Json`串解析位置的下一个`JsonToken`。

###JsonScope(解析上下文)

用来描述当前正在解析的位置所处的一个状态，为解析提供一个上下文做推断。比如，目前我们解析到了一个`{`字符，那么下一个字符串必须是空或者'"'，如此我们可以推断这个`"`肯定是一个JsonToken.Name(属性名)。此外，JsonScope还可以描述一个JsonArray或者JsonObject目前解析到的位置是否是第一个元素。`JsonScope`被存放在一个栈中，而且是成对出现。

	 private JsonScope[] stack = new JsonScope[32];
	 {
       push(JsonScope.EMPTY_DOCUMENT);  //静态初始化时push空Json文件到栈顶
     }
     需要注意的是，这里这个栈之只实现了一个push操作，并提供了动态增长，出栈操作只需要简单的简单的stacksize--
     

##JsonReader工作流
    
`JsonReader`的构造函数接受一个`Reader`的参数，它是一个`Json`串的`InputStream`包装出来的`Reader`。

对于这个上文`Json`文档的解析，`JsonReader`的理逻辑如下：

	List<Message> result = new ArrayList<Message>(); 
	jsonReader.beginArray();   //找到数组
	while(jsonReader.hasnext) {
		Message message = new Message();
		jsonReader.beginObject();   //开始数组中的对象
		while(jsonReader.hasnext) {
			String name = jsonReader.nextName();
			if(name.equals("id")) {
				message.setId(jsonReader.nextString());
			}else if(name.equals("text")) {
				message.setText(jsonReader.nextString());
			}else if(name.equals("geo")&&jsonReader.peek()!=NULL) {
				 List<Double> list = new ArrayList<Double>();
				 jsonReader.beginArray();
				 while(jsonReader.hasnext()) {
				 	list.add(jsonReader.nextDouble());
				 }
				 JsonReader.endArray();
			}
			jsonReader.beginObject();
		   .....开始解析User对象......
		}
		result.add(message);
	}
	
###一些函数的解释

####begin和 end系列函数
其实这两个函数只是从业务的角度出发，封装了一下，实际两个函数都是调用`except(JsonToken)`，作用是寻找`json`字符串中的下一个`JsonToken`。以`beginObject()`函数来说，它直接调用`except(JsonToken.BEGIN_OBJECT)`。

	private void expect(JsonToken expected) throws IOException {
    peek();  		//找到下一个JsonToken，赋值给JsonReader的token属性
    if (token != expected) {  //判断下一个Token是否等于指定的Token类型
      throw new IllegalStateException("Expected " + expected + " but was " + 		peek() + " at line " + getLineNumber() + " column " + 		getColumnNumber());
    }
    advance(); 	//下一步操作
	}
	
####peek函数

peek函数是JsonReader中最重要的函数，它的主要作用是进行上下文的判断，判断下一个JsonToken的读取方法。

下面看一下`peek()`函数的实现：

	public JsonToken peek() throws IOException {
	//之前解析出来的token还没有被消费，直接取回
    if (token != null) {
      return token;
    }
    //查看栈顶的JsonScope，这里可以看到JsonScope就是一个上下文的角色
    switch (stack[stackSize - 1]) {
    case EMPTY_DOCUMENT:	//json文档的开始 
      if (lenient) {	//json防止攻击引入的前导，防止跨域攻击
        consumeNonExecutePrefix();
      }
      stack[stackSize - 1] = JsonScope.NONEMPTY_DOCUMENT;   //注意这里不是push而是直接替换栈顶元素
      JsonToken firstToken = nextValue();   //返回流中的下一个开始JsonToken，在上个例子中就是'['，也就是这里返回的是一个JsonToken.BEGIN_ARRAY,这个值还会被保存在token属性中
      if (!lenient && token != JsonToken.BEGIN_ARRAY && token != JsonToken.BEGIN_OBJECT) {   //一个json文档开始无非就是一个对象或者数组
        throw new IOException("Expected JSON document to start with '[' or '{' but was " + token
            + " at line " + getLineNumber() + " column " + getColumnNumber());
      }
      return firstToken;
    case EMPTY_ARRAY:
    	//目前处于一个数组中，但是还没有开始读取任何数据
      return nextInArray(true);
    case NONEMPTY_ARRAY:
      return nextInArray(false);
    case EMPTY_OBJECT: //目前处于一个JsonObject中但是还没有读取任何数据
      return nextInObject(true);
    case DANGLING_NAME: //目前处于一个属性名的位置
      return objectValue();
    case NONEMPTY_OBJECT:
      return nextInObject(false);
    case NONEMPTY_DOCUMENT:
    	//当前处于一个文档的解析上下文中，可以理解为根元素下一级的环境
      int c = nextNonWhitespace(false);
      if (c == -1) {
        return JsonToken.END_DOCUMENT;
      }
      pos--;
      if (!lenient) {
        throw syntaxError("Expected EOF");
      }
      //找下一个JsonToken
      return nextValue();
    case CLOSED:
      throw new IllegalStateException("JsonReader is closed");
    default:
      throw new AssertionError();
    }
    }
    
    
####nextIn**()
 
 提供了数组、JsonObject的解析方法。
    
####nextValue()
	找下一个JsonToken
	
	private JsonToken nextValue() throws IOException {
    int c = nextNonWhitespace(true);	//找到当前解析位置之后的第一个非空白字符
    switch (c) {
    case '{':
      push(JsonScope.EMPTY_OBJECT);   //进入一个JsonObject
      return token = JsonToken.BEGIN_OBJECT;

    case '[':
      push(JsonScope.EMPTY_ARRAY);   //进入一个数组
      return token = JsonToken.BEGIN_ARRAY;
      //以上两种情况说明是一个新的结构的开始，所以需要pushJsonScope到栈中
    case '\'':
      checkLenient(); // fall-through
    case '"':
      //代表一个Json字符串值
      value = nextString((char) c);  //找到两个引号之间的String值
      return token = JsonToken.STRING;

    default:
      pos--;
      return readLiteral();
    }
    }
    
###JsonReader总结

JsonReader可以看作一个最基本的Json解析的接口，JsonReader中通过一个上下文来保存当前解析的环境，通过next**系列函数来获取下一个JsonToken。

#序列化--JsonWriter

理解了JsonReader的源码之后，再来看Writer就相对来说简单多了。现在我们有一个数组，List<Message>，它的值就如上面的例子，那如何序列化呢?

	public void writeJsonStream(OutputStream out, List<Message> messages) throws IOException {
          JsonWriter writer = new JsonWriter(new OutputStreamWriter(out, "UTF-8"));
          writer.setIndent("\t");			//设置每一行的缩进
          writeMessagesArray(writer, messages);
          writer.close();
        }

        public void writeMessagesArray(JsonWriter writer, List<Message> messages) throws IOException {
          writer.beginArray();   //[
          for (Message message : messages) {
            writeMessage(writer, message);
          }
          writer.endArray();
        }
       
        public void writeMessage(JsonWriter writer, Message message) throws IOException {
          writer.beginObject(); //{
          writer.name("id").value(message.getId()); // "id":23435356
          writer.name("text").value(message.getText()); //"text":"dflsfldsfljd"
          if (message.getGeo() != null) {
            writer.name("geo");
            writeDoublesArray(writer, message.getGeo());
          } else {
            writer.name("geo").nullValue();
          }
          writer.name("user");
          writeUser(writer, message.getUser());
          writer.endObject();
        }
       
        public void writeUser(JsonWriter writer, User user) throws IOException {
          writer.beginObject();
          writer.name("name").value(user.getName());
          writer.name("followers_count").value(user.getFollowersCount());
          writer.endObject();
        }
       
        public void writeDoublesArray(JsonWriter writer, List<Double> doubles) throws IOException {
          writer.beginArray();
          for (Double value : doubles) {
            writer.value(value);
          }
          writer.endArray();
        }
        


##JsonWriter中的函数
   
   事实上，可以看到JsonWriter的工作和JsonReader刚好相反，两个类的对json字符串的处理方式也基本相同。下面说一些业务流程中较为重要的方法。

###beginArray and beginObject

开始向流中写入一个数组或者JsonObject的开始，注意，这里不仅仅是写入一个`[`或者一个`{`这么简单，首先会调用`writeDeferredName`方法，它的主要功能是如果这个开始的JsonElement是JsonObject中的一个属性，那么它的前面肯定有一个和上一个元素的分割符号和一个名字。

###endArray and endObject

结束一个数组和对象。





