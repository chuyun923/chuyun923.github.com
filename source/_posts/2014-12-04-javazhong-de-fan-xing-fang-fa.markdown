---
layout: post
title: "java中的泛型方法"
date: 2014-12-04 16:36:56 +0800
comments: true
categories: Java
---
在通过反射获取类的私有域时希望写一个方法，可以指定一个域的名字和域的类型，可以返回指定类型的对象。
<!--more-->
方法实现如下：

    /**
     * 获取一个类的私有域
     * @param fromClass 私有域所在的类
     * @param fieldName 域的名字
     * @param <T>
     * @return
     */
    private <T> T getPrivateFiled(Class<T> fromClass, String fieldName) {
        T result = null;
        try {
            Field field = fromClass.getDeclaredField(fieldName);
            field.setAccessible(true);
            result = (T) field.get(this);
        } catch (NoSuchFieldException e) {
        } catch (IllegalAccessException e) {
        }
        return result;
    }
    
泛型方法的结构如下：<br />
![image][1]
<br />
值得说明的是，声明泛型方法首先需要在返回值前面用`<T>`表明这是一个泛型方法，持有一个泛型。当然，一般来说我们需要在参数中传入一个class对象，不然函数的泛型没有任何意义了。。。
[1]: http://images.cnblogs.com/cnblogs_com/CSU-PL/637624/o_java%e6%b3%9b%e5%9e%8b%e6%96%b9%e6%b3%95.png java泛型说明