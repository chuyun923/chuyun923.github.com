---
layout: ppt
title: "Java泛型"
date: 2015-05-04 10:14:20 +0800
comments: true
categories: Java
---
#Java基础知识点之泛型
##关键词

##泛型基础
###神马是泛型
便于程序员编写的一份代码可以用于多个类型，希望类或者方法具有通用的表达能力。
##泛型进阶
## 通配符
### 为什么需要 
例子：

	public class Parent { }
	public class Child1 extends Parent { }
	public class Child2 extends Parent { }
	
	List<Parent> parents = new ArrayList<Parent>();
	List<Child1> child1s = new ArrayList<Child1>();
	
`parents` 和 `child1s`能相互赋值么？

	parents ＝ child1s;	//no,parents.add(child2Object)，就破坏了child1s的安全性
	child1s = parents;	//no,child1s.get()有可能得到非Child1类型对象，就破坏了child1s的安全性
	
为什么有时候需要这种表达呢？考虑如下情况：

	public void printElements(List<Parent> elements){
		for(Parent o : elements){
			//假设，在Parent中定义了getSomeValue方法
			System.out.println(o.getSomeValue());
   		}
	}

我们需要打印所有`Parent`类型集合中的元素，但是我们不能将 `child1s`作为实参传送给`printElements(child1s) //inlegal`

###怎么使用
通配符的使用方式一共有三种：

	List<?>           listUknown = new ArrayList<Parent>();
	List<? extends Parent> listUknown = new ArrayList<Parent>();
	List<? super  Parent> listUknown = new ArrayList<Parent>();
	
List<?>表示集合中的类型未知，但是可以确定的是它们都是一个类型，为了保证安全性，List<?>只可以进行读操作。




##数组协变

所谓协变，指的是 子类sub的数组sub[]是父类super的数组super[]的子类型。
	
	Object[] objs = new String[1];
	
 

