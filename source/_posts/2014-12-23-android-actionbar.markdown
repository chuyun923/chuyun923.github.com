---
layout: post
title: "Android ActionBar"
date: 2014-12-23 10:58:00 +0800
comments: true
categories: Android
---
ActionBar作为Android3.0系统引入的一个重要的android app的风格，了解它的使用方法是每一个android开发者都必须要掌握的，本文主要基于android官方文档，对一些不常用的内容进行了删减，整理而成。
<!--more-->
说起3.0以后的ActionBar，其实在做3.0以前开发的童鞋应该有印象，Android提供了一个TitleBar，只不过默认的TitleBar只能显示一个标题和一个图标。它的样子大概如下：<br />
![][1]

如果需要自己定义一个更加炫酷的Titlebar,比较麻烦，可以参考这篇文章：[**自定义TitleBar**](http://www.cnblogs.com/a284628487/archive/2013/06/07/3125265.html)

ActionBar是TitleBar的增强版本，主要包括四个部分：

1. 为app提供一个唯一图片标识和导航的button,如果当前页面不是应用的首页，应该在app icon的左边添加返回的剪头。
2. 切换界面上显示的view，官方名称叫view control。
3. 放置一些显式的按钮，长按会显示它的title
4. 放置一个action overflow

对应如下图片：

![][2]

正常的ActionBar会占据一定的屏幕空间，在ActionBar显示或者隐藏时，屏幕中的数据都会重绘去适应。如果我们需要ActionBar浮动在内容上面，可以将`windowActionBarOverlay`属性设置为true。

##增加ActionBar的Item
在android菜单中曾经提到如何在ActionBar中增加菜单项，这里不再赘述。这里补充一个小的case：

 在ActionBar上显示的item也可以同时显示icon和title，当然前提是ActionBar的空间富余。通过在showAsAction中增加withText。

##分离的ActionBar(split)
在分辨率不大的设备上，如果有足够多的Item需要显示，可能面临着屏幕宽度不够而显示不了那么多的Icon。分离ActionBar中的logo和菜单项，将菜单项移动到屏幕的下面，如图：


![][3]

如果需要系统处理，那么我们需要指定属性：` uiOptions="splitActionBarWhenNarrow"`，但是如果需要支持低版本，我们可以这样写：

		<manifest ...>
    		<activity uiOptions="splitActionBarWhenNarrow" ... >
        		<meta-data android:name="android.support.UI_OPTIONS"
                   android:value="splitActionBarWhenNarrow" />
    		</activity>
		</manifest>
		
##ActionBar的导航button
如果设置ActionBar.setDisplayHomeAsUpEnable(true),会产生一个导航按钮，如图所示：

![][4]

这个按钮和系统的back按键的区别在于，系统的back键有时候仅仅是为了返回到屏幕的上一次状态，而点击这个导航键就会返回到上一次的界面。
如果仅仅set这个导航键可见，它并不会响应任何操作，我们需要手动去指定点击这个导航键将会打开那个Activity，有两种方法可以来指定：

1. 在Menifest文件中通过activity的parentActivityname属性指定，当然这个属性是4.1加入的，如果要兼容之前的版本，可以通过meta的方式来做到。
2. 在Activity中复写getParentActivityIntent，可以指定在app内导航的Intent。在别的应用中打开app，可以通过复写onCreateNavigateUpTaskStack来实现。

注意，在support v7包中的actionbarActivity中，对以上提供了默认实现。

##ActionView
actionview是在点击action button之后，一个显示出来的交互view,如下图：

![][5]


可以通过Menu item的`actionViewClass`来指定一个widget。我们也可以直接自己指定一个自定义布局给Action button，只需要把layout的id赋值给`actionLayout`即可。

app要获取这个Action View中的一些事件，可以在onCreateOptionsMenu中通过`menuItem.getActionView()`来获得。

下面来说一个比较有意思的可折叠的actionview，前文可知，如果设置actionLayout，那么如果ActionBar上面空间足够时，会直接把这个自定义的layout显示出来，但是如果这个item被放到overflow里面去了呢？此时`showAction`中的一个属性`collapseActionView`的作用就会显示出来，这个属性的作用是：

如果这个action button被放到overflow中，点击后还是可以把它的actionview打开(可折叠)，如果不加这个属性，那么系统将不会打开actionview。如果是打开折叠的actionview，ActionBar会自动把导航剪头加上，并且清空ActionBar上面的action button(除去overflow按钮)，如下图：

![][6]

我们可以在`onCreateOptionsMenu`中  通过`MenuItemCompat.setOnActionExpandListener`来监听actionview的扩张事件。

##Action Provider
上一节提到的actionview，它是直接在ActionBar上面展示出来，ActionProvider是在ActionBar上展示一个下拉的菜单，如下图：

![][7]

创建一个Action provider可以通过指定给`android:actionProviderClass`属性一个ActionProvider即可。

##添加导航的tab
这种方式已经不被推荐使用了，所以这里不做详细解释，使用它很简单。

##下拉导航
下拉导航是在ActionBar的左边提供一个action provider类似的下拉框，可以使用在改变界面内容的频率不频繁的情况下可以使用Drop-down Navigation来进行切换选项的提供，如下图：

![][8]

使用下拉导航有几个步骤：

1. 创建一个SpinnerAdapter，提供下拉项的布局
2. 实现一个选择接口 `ActionBar.OnNavigationListener`，处理事件
3. 在Activity的onCreate里面调用setnavigationMode()
4. actionbar.setListNavigationCallbacks(spinneradapter,mnavigationcallback),将1和2定义的东西传进去

[1]:http://androidpeople.files.wordpress.com/2009/12/withicon.jpg
[2]:https://developer.android.com/design/media/action_bar_basics.png
[3]:https://developer.android.com/images/ui/actionbar-splitaction@2x.png
[4]:https://developer.android.com/images/ui/actionbar-up.png
[5]:https://developer.android.com/images/ui/actionbar-searchview@2x.png
[6]:http://images.cnblogs.com/cnblogs_com/CSU-PL/637624/o_5B4AFB05-1AFC-4F4F-887F-304187E96E16.png
[7]:https://developer.android.com/images/ui/actionbar-shareaction@2x.png
[8]:http://developer.android.com/images/ui/actionbar-dropdown@2x.png
