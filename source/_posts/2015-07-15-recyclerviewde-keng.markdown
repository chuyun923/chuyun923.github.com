---
layout: post
title: ""RecyclerView中LayoutManager的坑"
date: 2015-07-15 16:41:58 +0800
comments: true
categories: Android
---
##问题描述

使用`RecyclerView`版本时，报如下异常：

	java.lang.NullPointerException: Attempt to invoke virtual method 'void android.support.v7.widget.RecyclerView$LayoutManager.stopSmoothScroller()' on a null object reference
	
查看堆栈，是在Activity的Destory方法一路调用到`RecyclerView`的`onDetachedFromWindow`里面报的NPE。

##原因

Activity在销毁时，回去调用View中的`onDetachedFromWindow`,搂一眼代码：

	 @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        if (mItemAnimator != null) {
            mItemAnimator.endAnimations();
        }
        mFirstLayoutComplete = false;

        stopScroll();
        mIsAttached = false;
        if (mLayout != null) {
            mLayout.onDetachedFromWindow(this, mRecycler);
        }
        removeCallbacks(mItemAnimatorRunner);
    }

	 public void stopScroll() {
        setScrollState(SCROLL_STATE_IDLE);
        stopScrollersInternal();
    }

    /**
     * Similar to {@link #stopScroll()} but does not set the state.
     */
    private void stopScrollersInternal() {
        mViewFlinger.stop();
        //什么！！！！
        mLayout.stopSmoothScroller();
    }

`stopScrollersInternal`中没有判空。

##其他问题
其实这个问题还有另外一个版本，如果在XML中定义了一个`RecyclerView`,而不调用`setLayoutManager`,会直接crash，NPE报得飞起。大概是在`OnMeasure`里面直接调用了`LayoutManager` 的方法导致的，问题也是同一个，木有判空。
人与人之间基本的信任呢？我在xml里面定义一个view，不在代码里面执行指定操作就crash，略屌。。。

##解决方案
[stackoverflow](http://stackoverflow.com/questions/26702633/why-am-i-getting-a-null-reference-on-my-recyclerview/26908738#26908738)
让我们继承RecyclerView，然后重写方法。

不过我发现在最新的support包里面，也就是RecyclerView的5.1.0_r1版本已经修复了这个问题,google大法好！

对于上面提到第二个问题，如果在`onMeasure`的时候`LayoutManager`为空，那么`RecyclerView`会给一个默认值。

所以你需要做的仅仅是升级到最新版本即可。

`compile 'com.android.support:recyclerview-v7:22.2.0'` 
