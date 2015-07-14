---
layout: post
title: "Gradle指南-11"
date: 2015-02-05 00:55:29 +0800
comments: true
categories: Android
---
#使用命令行执行gradle任务
[指南原文，看这里] [1]

gradle执行task的基本命令是：

`gradle taskname`

**gradle支持模糊匹配，taskname只要能够模糊唯一匹配即可**

##gradle执行多个task
命令格式： `gradle task1 task2 ...`

这样可以按顺序执行task1，task2，以及这些task所依赖的task,一个task及其依赖执行完了，才会执行下一个task。task依赖最深的会被最先执行然后递归出来。**每一个task不管被依赖了几次，在一条gradle命令时，都只执行一次。**

例有四个task的依赖如下图：

![][2]

执行`gradle dist test`，首先会执行`dist`的依赖，所以执行顺序为：
compile-->compileTest-->test-->dist(每个task都只执行一遍)

##排除task
如果不想执行某个task(一般是task依赖的task)，可以通过：

`gradle task1 task2 -x excludingTask1,excludingTask2..`，去掉的task以及它所带入的依赖task都不会被执行。

##gradle tasks... --continue
如果不加，那么gradle在执行task的时候，如果遇到错误，那么以下所有没有执行的task都会终止，这一次build结束。加上continue参数之后，每一次gradle命令都会执行所有的task，这样可以检查所有的task。
**如果一个task失败，那么所有基于这个task的task都不会被执行。**


-------
[1]:http://gradle.org/docs/current/userguide/tutorial_gradle_command_line.html
[2]:http://gradle.org/docs/current/userguide/img/commandLineTutorialTasks.png