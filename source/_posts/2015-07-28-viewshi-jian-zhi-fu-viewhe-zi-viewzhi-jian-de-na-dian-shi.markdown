---
layout: post
title: "View事件传递之父View和子View之间的那点事"
date: 2015-07-28 09:29:14 +0800
comments: true
categories: Android
---
#View事件传递之父View和子View之间的那点事

`Android`事件传递流程在网上可以找到很多资料，`FrameWork`层输入事件和消费事件，可以参考：

1. [Touch事件派发过程详解] [1]

这篇blog阐述了底层是如何处理屏幕输，并往上传递的。`Touch`事件传递到`Activity`的`DecorView`时，往下走就是`ViewGroup`和子`View`之间的事件传递，可以参考郭神的这两篇博客

1. [Android事件分发机制完全解析，带你从源码的角度彻底理解(上)] [3]
2. [Android事件分发机制完全解析，带你从源码的角度彻底理解(下)] [4]

郭神的两篇博客清楚明白地说明了`View`之间事件传递的大方向，但是具体的一些晦暗的细节阐述较少，本文主要是总结这两篇博客的同时，侧重于两点：

1. 事件分发过程中一些细节到底如何实现的？
2. 子`view`到底如何和父`View`抢事件，父`View`又是如何拦截事件不发送给子`View`，以及如果我们需要处理这种混乱的关系才能让两者和谐相处？。


##MotionEvent抽象
要明白`View`的事件传递，很有必要先说一下`Touch`事件是如何在`Android`系统中抽象的，这主要使用的就是`MotionEvent`。这个类经历了几次重大的修改，一次是在2.x版本支持多点触摸，一次是4.x将大部分代码甩给`native`层处理。

###一次简单的事件

我们先举个栗子来说明一次完整的事件，用户触屏 滑动 到手机离开屏幕，这认为是一次完整动作序列(`movement traces`)。一个动作序列中包含很多动作`Action`，比如在用户按下时，会封装一个`MotionEvent`，分发给视图树，我们可以通过`motionevent.getAction`拿到这个动作是`ACTION_DOWN`。同样，在手指抬起时，我们可以接收到`Action`类型是`Action_UP`的`MotionEvent`。对于滑动(`MOVE`)这个操作，`Android`为了从效率出发，会将多个`MOVE`动作打包到一个`MotionEvent`中。通过`getX getY`可以获取当前的坐标，如果要访问打包的缓存数据，可以通过`getHistorical**()`函数来获取。

###加入多点触摸

对于单点的操作来看，`MotionEvent`显得比较简单，但是考虑引入多点触摸呢？我们定义一个接触点为(`Pointer`)。我们从`onTouch`接受到一个`MotionEvent`，怎么拿到多个触碰点的信息？为了解开笔者刚开始学习这部分知识时的困惑，我们首先树立起一种概念：一个`MotionEvent`只允许有一个`Action`(动作)，而且这个`Action`会包含触发这次`Action`的触碰点信息，对于`MOVE`操作来说，一定是当前所有触碰点都在动。只有`ACTION_POINTER_DOWN`这类事件事件会在`Action`里面指定是哪一个`POINTER`按下。


在`MotionEvent`的底层实现中，是通过一个16位来存储`Action`和`Pointer`信息(`PointerIndex`)。低8位表示`Action`，理论上可以表示255种动作类型;高8位表示触发这个`Action`的`PointerIndex`，理论上`Android`最多可以支持255点同时触摸，但是在上层代码使用的时候，默认多点最多存在32个，不然事件在分发的时候会有问题。

`MotionEvent`中多个手指的操作`API`大部分都是通过`pointerindex`来进行的，如：获取不同`Pointer`的触碰位置,`getX(int pointerIndex)`;获取`PointerId`等等。大部分情况下，`pointerid == pointeridex`。


`ACTION_DOWN` OR `ACTION_POINTER_DOWN`:

这两个按下操作的区别是`ACTION_DOWN`是一个系列动作的开始，而`ACTION_POINTER_DOWN`是在一个系列动作中间有另外一个触碰点触碰到屏幕。

这部分详细的描述，请参考：
[android触控,先了解MotionEvent][5]

到这里，铺垫终于结束了，我们开始直奔主题。


##View的事件传递
`Android`的`Touch`事件传递到`Activity`顶层的`DecorView`(一个`FrameLayout`)之后，会通过`ViewGroup`一层层往视图树的上面传递，最终将事件传递给实际接收的`View`。下面给出一些重要的方法。如果你对这个流程比较熟悉的话，可以跳过这里，直接进入第二部分。

###dispatchTouchEvent
事件传递到一个`ViewGroup`上面时，会调用`dispatchTouchEvent`。代码有删减

	public boolean dispatchTouchEvent(MotionEvent ev) {

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Attention 1 ：在按下时候清除一些状态
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                //注意这个方法
                resetTouchState();
            }

            // Attention 2：检查是否需要拦截
            final boolean intercepted;
            //如果刚刚按下 或者 已经有子View来处理
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                //  不是一个动作序列的开始 同时也没有子View来处理，直接拦截
                intercepted = true;
            }

			  //事件没有取消 同时没有被当前ViewGroup拦截，去找是否有子View接盘
            if (!canceled && !intercepted) {
					//如果这是一系列动作的开始  或者有一个新的Pointer按下 我们需要去找能够处理这个Pointer的子View
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    
                    //上面说的触碰点32的限制就是这里导致
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        
                        //对当前ViewGroup的所有子View进行排序，在上层的放在开始
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);
								
								  // canViewReceivePointerEvents visible的View都可以接受事件
								  // isTransformedTouchPointInView 计算是否落在点击区域上
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
								
								  //能够处理这个Pointer的View是否已经处理之前的Pointer，那么把
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }                           }
                            //Attention 3 : 直接发给子View
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
  
                        }
                    }

                }
            }

            // 前面已经找到了接收事件的子View，如果为NULL，表示没有子View来接手，当前ViewGroup需要来处理
            if (mFirstTouchTarget == null) {
                // ViewGroup处理
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
               
               	if(alreadyDispatchedToNewTouchTarget) {
               		               	//ignore some code
                		if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                 	}
               	}

            }
        return handled;
    }

上面代码中的***`Attention`***在后面部分将会涉及，重点注意。

这里需要指出一点的是，一系列动作中的不同`Pointer`可以分配给不同的`View`去响应。ViewGroup会维护一个`PointerId`和处理`View`的列表`TouchTarget`，一个`TouchTarget`代表一个可以处理`Pointer`的子`View`，当然一个`View`可以处理多个`Pointer`，比如两根手指都在某一个子`View`区域。`TouchTarget`内部使用一个`int`来存储它能处理的`PointerId`，一个`int `32位，这也就是上层为啥最多只能允许同时最多32点触碰。

看一下`Attention 3` 处的代码，我们经常说`view`的`dispatchTouchEvent`如果返回false，那么它就不能系列动作后面的动作，这是为啥呢？因为`Attention 3`处如果返回`false`，那么它不会被记录到`TouchTarget`中，ViewGroup认为你没有能力处理这个事件。

这里可以看到，`ViewGroup`真正处理事件是在`dispatchTransformedTouchEvent`里面，跟进去看看：

###dispatchTransformedTouchEvent

	private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {

		  //没有子类处理，那么交给viewgroup处理
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }
        return handled;
    }

可以看到这里不管怎么样，都会调用`View`的`dispatchTouchEvent`,这是真正处理这一次点击事件的地方。

###dispatchTouchEvent

		public boolean dispatchTouchEvent(MotionEvent event) {
			if (onFilterTouchEventForSecurity(event)) {
            //先走View的onTouch事件，如果onTouch返回True
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
      	  return result;
		}

我们给`View`设置的`onTouch`事件处在一个较高的优先级，如果`onTouch`执行返回`true`，那么就不会去走`view`的`onTouchEvent`，而我们一些点击事件都是在`onTouchEvent`中处理的，这也是为什么`onTouch`中返回true，`view`的点击相关事件不会被处理。

###小小总结一下这个流程

`ViewGroup`在接受到上级传下来的事件时，如果是一系列`Touch`事件的开始(`ACTION_DOWN`)，`ViewGroup`会先看看自己需不需要拦截这个事件(`onInterceptTouchEvent`，`ViewGroup`的默认实现直接返回`false`表示不拦截)，接着`ViewGroup`遍历自己所有的`View`。找到当前点击的那个`View`，马上调用目标`View`的`dispatchTouchEvent`。如果目标`View`的`dispatchTouchEvent`返回false，那么认为目标`View`只是在那个位置而已，它并不想接受这个事件，只想安安静静的做一个`View`(我静静地看着你们装*)。此时，`ViewGroup`还会去走一下自己`dispatchTouchEvent，Done！`

## 子View和父View的撕*大战

终于来到本文的重要环节，子View和父布局(`ViewGroup`)是如何撕逼的。我们经常遇到这样的问题：在`ListView`中放一个`ViewPager`不能滑动的问题，其实这里就会涉及到子View和布局之间的协商，事件处理到底你上还是我上。

首先需要明确一点的是，一个事件肯定是由`ViewGroup`传递给自己的子`View`的，所以`ViewGroup`具有绝对的权威来禁止事件往下传，这就是`onInterceptTouchEvent`方法。可以看上面`ViewGroup中的dispatchTouchEvent`的`Attention 1 `和` Attention 2`。

先看`Attetion2`：
进行判断有有两个条件：1，如果是一次新的事件 or  在一次事件中但是已经有子View来处理这个事件，那么父类需要去看看是否拦截这次事件。否则，直接拦截(此时处于一系列动作的中间，而且没有子view来接盘，那么ViewGroup就直接拦下来)。

决定是否拦截有两个步骤，

1.  `disallowIntercept` 是否驳回拦截，默认`false`。注意这个值是子`View`和撕*的关键，因为`ViewGroup`开放了给这个标记赋值的接口`requestDisallowInterceptTouchEvent()`，而且这个方法直接往上递归，这个`ViewGroup`的各级父容器都会设置驳回拦截。
2.  `onInterceptTouchEvent` 虽然`ViewGroup`中默认返回false，但是在很多有滑动功能的`ViewGroup`里面(如`scrollview ListView`等)会处理各种情况，决定是否拦截这个事件，所以就会出现之前说的`ListView`中的`Viewpager`不能滑动的问题，原因是事件被父View拦截了。

在`Attetion1`的位置如果是一次新的`ACTION_DOWN`，那么会把之前事件传递设置的各种状态清除。

### 对ViewGroup来说需要做什么

对于一个需要拦截事件的`ViewGroup`，它通常都有一些特殊的操作，比如`ScrollView`，比如`ViewPager`，它重写`onInterceptTouchEvent`是非常关键的，这也是能和子`View`和谐相处的关键。举个例子，我自己定义了一个`ViewGroup`：

	@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if(ev.getActionMasked() == MotionEvent.ACTION_DOWN) {
            return true;
        }
        return super.onInterceptTouchEvent(ev);
    }

这样会发生什么？

所有位于MyViewGroup中的子View收不到任何的事件，原因可以看一下`Attention2`的代码，判断是否拦截是在系列动作按下时会进行判断，如果此时拦截，那么直接不会去查找相应处理的子View，所以`touchtarget`为空，那么接下来的动作都直接被`ViewGroup`笑纳。

所以哪怕再强势的`ViewGroup`，一般都是在`Down`的时候给子类机会去掉用`requestDisallowInterceptTouchEvent`，如设置驳回拦截，那么在ViewGroup分发事件的时候，会跳过`onInterceptTouchEvent`的执行。

### 子View需要做什么

对于子View来说，在合适的时机调用`requestDisallowInterceptTouchEvent`即可。当然啥时候合适？对于一个`View`来说，那就是在`dispatchTouchEvent`或者`onTouchEvent`来调用。

对于`ViewGroup`来说，通常我们会在`onInterceptTouchEvent`进行判断。比如我们经常会遇到在`ListView`里面套了`ViewPager`导致`ViewPager`不能滑动的问题，通常的处理方式：

	 @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        if (absListView != null) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    mDownX = event.getX();
                    mDownY = event.getY();
						//ACTION_DOWN的时候，赶紧把事件hold住
                    getParent().requestDisallowInterceptTouchEvent(true);
                    break;
                case MotionEvent.ACTION_MOVE:

                    if(Math.abs(event.getX() - mDownX)>Math.abs(event.getY()-mDownY)) {
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }else {
                    	//发现不是自己处理，还给父类
                        getParent().requestDisallowInterceptTouchEvent(false);
                    }
                    break;
                case MotionEvent.ACTION_CANCEL:
                case MotionEvent.ACTION_UP:
                		//其实这里是多余的
                    getParent().requestDisallowInterceptTouchEvent(false);
            }

        }
        return super.onInterceptTouchEvent(event);
    }

##总结

本来打算写一个短篇的，结果一个不小心，弄成了长篇大论。

最后需要注意一点的是，所有我们上述讨论的内容都是在一层层递归中进行,而且`requestDisallowInterceptTouchEvent`这个函数也是递归调用的。

我们可以认为`ViewGroup`是一个具有绝对话语权但是从不专政的霸道总裁，它自己可以拦截处理某些事件，比如`Viewpager`的横滑，但是它也可以给子View足够的空间去要求这个事件给自己处理。作为一名开发者，一方面在自己定义`ViewGroup`时需要考虑能够给子View足够空间中断自己的拦截；一方面自己定义View时，我们需要在合适的时候跟父View索要事件。`ViewPager(新版)`作为容器来说，它需要拦截横滑事件，同时，自己具备了和父`View`争抢事件的能力，所以不管把`ViewPager`放到什么布局中，它都能正确处理。看看它的`onInterceptTouchEvent`怎么写的吧，完美的体现了这一思想。

[1]:http://blog.csdn.net/stonecao/article/details/6759189
[3]:http://blog.csdn.net/guolin_blog/article/details/9097463
[4]:http://blog.csdn.net/sinyu890807/article/details/9153747
[5]:http://my.oschina.net/banxi/blog/56421
