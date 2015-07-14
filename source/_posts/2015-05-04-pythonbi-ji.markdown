---
layout: post
title: "Python笔记"
date: 2015-05-04 22:05:04 +0800
comments: true
categories: Python
---
##Python语言简介
Python是一种解释执行的高级脚本语言。解释执行意味着Python代码的执行需要一个类似JVM的运行环境，与JVM类似，Python也存在多个不同的解析器(运行环境)，比如我们最常用的CPython，这是一个官方版本使用C编写的解析器。

Python的代码有两种执行方式

1. 交互式。直接在命令行输入python，进入交互模式，退出交互模式的方法有输入 `exit()` 或者 `Ctrl+D`
2. 源文件。使用`.py`后缀来保存python脚本。

##源文件组织行
在Linux/Unix环境中，我们可以在源文件的第一行指定一个注释组织行，指定当前系统执行程序的解析器的位置。

	#!/usr/bin/python    //#!开头
	
同时，我们给文件增加 `x` 执行权限

	chmod a+x *.py
	
如此一来，我们可以直接输入文件名来执行程序，这样指定的源程序的后缀不一定要是`.py`

##基本类型
###数据类型

1. 整数，长整数
2. 浮点数
3. 复数

###字符串
使用方法和转义字符基本上和其他语言差不多，字符串也是不可改变的。
###布尔值
True or False  注意大小写 
提供了 `and or not`运算。
###变量
Python对于变量的命名和Java一样，但是Python和JS一样，是动态语言，即变量可以被多次赋值，而且可以每次赋值的类型不一样。
##语法
###逻辑行和物理行
物理行是你在编写程序时所看见的。逻辑行是`Python`看见的单个语句。`Python`假定每个物理行对应一个 逻辑行。如果一个逻辑行需要占用多个物理行，那么可以通过`\`来进行连接。如：

	print \
	i
	
###缩进
空白在python语法中占有较为重要的位置，它用来作为代码块分组的选择。

###运算符
下面仅列出和`Java`不一样的地方

1. ** 幂  x的y次幂
2. // 整除，取整数部分
3. `not and or ` 对应 `! && ||`

###if
	
	if boolean:
		some code
	elif boolean:   //else if
		some code
	else:
		some code
		
 