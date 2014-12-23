---
layout: post
title: "Android之菜单"
date: 2014-12-19 17:19:36 +0800
comments: true
categories: Android
---

本文参考自官方文档：https://developer.android.com/guide/topics/ui/menus.html


Android为了维护app之间一个统一的操作习惯，提供了Menus来处理用户和Activity之间的一些交互。但是在不同的系统版本上面推荐的Menu不一样。比如在android 3.0以下，由于Google会要求所有设备生产商提供一个菜单的实体键，所以在3.0一下菜单的主要弹出方式就是点击菜单实体键，弹出6个条目的菜单面板。在3.0以后，引入ActionBar，打开菜单行为转变成点击ActionBar上面的overflow按钮。这两种菜单面板的操作一般都是影响到整个app的操作。
<!--more-->
下图是3.0一下的Option Menu的样子：<br />


![][1]

这是3.0上的ActionBar的样子：

![][2]


当然除了上面提到的菜单面板，Android还提供了上下文菜单(Context menu)和弹出菜单(Popup Menu)。尽管存在三种不同的菜单，但是Android提供了一个统一操作的API。

##在XML中定义菜单项
Menu文件定义的位置在 res/menu目录下，而且顶层元素必须是Menu。在Menu中可以放置item和group，其中group又可以包含多个item。在group中的所有item都会共享某一些熟悉，主要是用来对group进行分类管理,比如设置某一类菜单项的可见。

多级菜单可以通过在item中嵌入Menu来现实。

###选项菜单
前面已经说到，选项菜单在3.0前后版本存在一些差别。在Fragment和Activity中都可以创建选项菜单,系统会首先显示Activity中创建的item，然后按照fragment加入的顺序添加item。**在3.0一下时，onCreateOptionsMenu是在点击菜单键时触发；在3.0以上，则是在Activity创建时就会调用。**

响应点击事件，可以在onOptionsItemSelected中进行。注意这个事件处理函数需要返回一个boolean值。如果已经处理了这次点击需要返回true，否则直接调用super.onOptionsItemSelected()(返回fasle)。这个事件的处理流程是，事件先被送到Activity，然后按照先进先达的顺序，直至莫一个fragement处理了这次点击或者所有的fragment都已经遍历了。

可以在菜单的item中指定`android:onClick`，这个点击事件的处理函数必须是Activity中签名为public 并且接受一个MenuItem的参数。

**更新菜单中的选项**，我们可以通过onCreateOptionsMenu来创建菜单项，但是如果想在运行时改变菜单中的选项，可以重写onPrepareOptionsMenu方法来实现。在3.0以下，这个方法会在菜单键每次按下的时候触发；在3.0以上，由于ActionBar是一直显示的，所以我们如果需要改变菜单，可以主动调用`invalidateOptionsMenu()`,然后系统会去走onPrepareOptionsMenu。

###上下文菜单
Contextual Menu主要多用于和界面特别是AdapterView中的某一个item进行交互,通过长按控件来呼出一个Action Menu。如果说Option menu(Actionbar)上面的菜单选项是针对整个app的范围，那么Contextual Menu从名字就可以看出来是针对当前context范围内的操作。存在两种方式：

+ floating context menu，通常在长按ListView中的某一项之后出现。
+ Contextual action mode，会显示出一个ActionBar，它是一种ActionMode的实现，在3.0系统以上才能使用。

下图左边是floating context Menu，右边是Contextual action mode

![][3]

####floating context menu
floating context menu是3.0一下版本建议使用的，针对当前Context的一个操作面板，通过长按指定控件呼出。**长按事件如果也被监听，那么会先执行长按事件，再执行onContextItemSelected。如果长按事件处理返回true，那么context Menu不会被呼出**：


1. 通过Activity或者Fragment的registerForContextMenu,传入一个View，为它注册一个floating context menu
2. 实现onCreateContextMenu，来创建Menu条目
3. 实现onContextItemSelected处理选择某一个菜单的事件

同样需要注意的是在处理事件时，也是由Activity最先接收，然后按照加入顺序分发到每一个Fragment。

####contextual action mode
从上图可以看到Contextual action和ActionBar颇有几分相似，但是两者直接并无直接关联。在3.0一下版本中，我们需要使用support包中的兼容方案。
单个View使用步骤如下：

1. 实现ActionMode.CallBack，这里主要实现action mode的主要逻辑
2. 在合适的时候通过Activity或者fragment的startSupportActionMode来进入一个contextaul action mode。

在AdapterView中使用，步骤如下：

1. 实现AbsListView.MultiChoiceModeListener，其实这个玩意继承自ActionMode.CallBack，增加了onItemCheckedStateChanged函数，用来处理当AbsListView中某一项的选择状态改变时的操作。
2. 通过设置AbsListView的MultiChoiceModeListener，注意，这里默认进入Action Mode的动作是长按某一个Item。

###Popup Menu
API level 11加入。在android中还提供了一种用来相对于界面上已经存在的一个控件的菜单，比如类似于ActionBar上面的overflow。

![][4]

使用Popup Menu的步骤如下：

1. 通过context和相对的View创建PopupMenu对象。
2. 通过popupMenu对象来设置Menu的布局和监听事件函数。

**需要注意的是PopupMenu弹出的位置是自适应的，主要看这个View在那个地方有空间，就会在哪个方向上面弹出来。**

###选择菜单项
选择菜单和前面所提及的菜单类型不同，它仅仅是一种菜单项的表现形式。在本文之前提及的所有菜单中，每一个菜单项的呈现方式都是简单的文字(或者icon),如果我们要加入一种单选框或者复选框的效果，可以使用item的checkable属性。效果如下图：

![][5]

需要注意的是，在option Menu中，如果一个菜单项是以icon的方式显示出来，那么它将不会显示选择框。

我们亦可以通过group来为一组item设置选择条件，这才是它本来的意义。`android:checkableBehavior`可以设置成radio 或者checkbox或者none，默认应该是checkbox。
**选择菜单项是不能保存状态的，如果app退出，下次再进入状态就不存在了。**

###意图菜单项
如果我们需要在菜单项中通过Intent启动另外一个Activity，Menu提供了专门的类型来处理，这就是Intent Options。而且意图菜单项还可以在系统中解析这个Intent是否能够被Activity了解，如果系统中没有Activity能够接受这个Intent，那么这个意图菜单项将不会展示出来。使用步骤如下：

1. 在定义的Intent中增加一个category：` CATEGORY_ALTERNATIVE and/or CATEGORY_SELECTED_ALTERNATIVE`
2. 调用Menu.addIntentOpions()。

addIntentOptions方法返回增加的item数目，所以通过Intent解析的item都会被加入菜单中。**这个菜单项的item title就是intent-filter的android:label,icon是application icon**

感觉这个东东和settings等系统模块一样，在中国开发者的眼中根本不会去用它，所以老外才会感叹中国做的app怎么这么难用，每个应用的风格都不一样。。。


[1]:https://developer.android.com/images/options_menu.png
[2]:https://developer.android.com/images/ui/actionbar.png
[3]:https://developer.android.com/images/ui/menu-context.png
[4]:https://developer.android.com/images/ui/popupmenu.png
[5]:https://developer.android.com/images/radio_buttons.png