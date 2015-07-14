---
layout: post
title: "Android 视图树状态保存"
date: 2015-07-13 18:13:40 +0800
comments: true
categories: Android
---
#Android 视图树状态保存
##FragmentTabHost引起的思考
公司的项目是一个标准的`FragmentTabHost`与`Fragment`构成的四TAB布局。其中三个TAB中都包含有`ListView`来展现一个列表。用户在切换TAB时，`ListView`的当前位置会自动被保存，切换回来之后会自动滚动到上次的位置。

我们知道`FragmentTabHost`内部对于`Fragment`的切换使用的是`attach`和dettach，此时必然会走Fragment的视图树的重建，也就是在切换tab的时候，fragment的UI元素会进行重建，当然也包括重建其中的ListView。那为什么在重建之后Listview能找到正确位置？为了找到解释，我开始查阅代码。
<!--more-->
第一个想法是否是我们通过`Fragment`的`onSaveInstanceState`和`Bundle`来保存的呢？但是看`Fragment`中的方法并没有任何代码进行保存，也没有进行恢复。

有人告诉我是通过`Adapter`来进行保存的，因为我们的`Fragment`会持有`listview`使用的`ListAdapter`，所以在重建的时候`listadapter`实际上复用之前的`adapter`，但是，通查adapter的代码，发现并没有任何保存当前位置的代码。

这样不科学的事情怎么会发生？一个新创建出来的ListView居然会保存之前一个ListView的状态？

##Android View状态保存
我们经常有这样的经验，在一个`EditText`中我们输入一些文字，在屏幕翻转时，`EditText`进行重建，其中的文字还会保留(前提是你为这个`EditText`指定了ID)。
解释这个问题，我们需要从`Android`的`View`状态保存机制说起。

在`View`中`Android`定义了`saveHierarchyState`，它用来保存这个`View`所在的视图树的状态，传进来的参数是父`view`保存的状态，看到这里，你应该已经意识到了这又是`android View`中常用的递归伎俩。

	    public void saveHierarchyState(SparseArray<Parcelable> container) {
        	dispatchSaveInstanceState(container);
    	 }
    	 
    	protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
    	  //这里可以看到如果你不指定view的ID，系统是不会给你保存view的状态的
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
            //获取当前View需要保存的Parcelable
            Parcelable state = onSaveInstanceState();
            if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                throw new IllegalStateException(
                        "Derived class did not call super.onSaveInstanceState()");
            }
            if (state != null) {
                // Log.i("View", "Freezing #" + Integer.toHexString(mID)
                // + ": " + state);
                
                //妈蛋，直接是简单粗暴的 viewId ----> Parcelable的映射
                container.put(mID, state);
            }
        }
    }
    
从上面的代码，我们可以大致想象一下`saveHierarchySate`的过程，顶层的`view`调用`saveHierarchyState`,然后`dispatchSaveInstanceState`保存自己，为了让递归走下去，我们都能想象到在`ViewGroup`中的`dispatchSaveInstanceState`会调用子`View`的`saveHierachySate`，是不是这样呢，搂一眼`ViewGroup`中的代码：

	@Override
    protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
    	//先把自己放到SparseArray中
        super.dispatchSaveInstanceState(container);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        //保存孩儿们的状态
        for (int i = 0; i < count; i++) {
            View c = children[i];
            if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
                c.dispatchSaveInstanceState(container);
            }
        }
    }

看到这里，相信你已经迫不及待想要看顶层`View`是在什么时机保存视图树信息的，Activity中的`onSaveInstanceState`有如下语句：

	outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
	
这里调用了跟到`PhoneWindow.saveHierarchyState()`,

	SparseArray<Parcelable> states = new SparseArray<Parcelable>();
    mContentParent.saveHierarchyState(states);
    
这里交代地很清楚明白。ok，总结一下这个流程

`Acitvity`在状态需要保存时，直接new了一个`SparseArray`，建立m`ovieId`和`Parcelable`的映射，其中每一个`View`都会使用一个`Parcelable`来序列化需要保存的内容，`ViewGroup`在保存了自己的同时，也会去调用子类的保存状态的方法，并把上面传下来的`SparseArray`一直往下传。

## 视图树状态恢复

**视图树保存下来的SparsesArray不是被ViewGroup自己持有(避免View被重新new出来，这些信息丢失，这也是前面疑惑的根源)。**以`Activity`为例，我们可以看到整个视图树状态保存之后会放到一个`Bundle`中，那我们来看看视图树是如何恢复的。

在`Activity`的`onRestoreInstanceState`中

	    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        	if (mWindow != null) {
            Bundle windowState = savedInstanceState.getBundle(WINDOW_HIERARCHY_TAG);
            if (windowState != null) {
                mWindow.restoreHierarchyState(windowState);
            }
        	}
    	}
    	
跟到`PhoneWindow`中的`restoreHierarchyState`
 
 		 
		SparseArray<Parcelable> savedStates = savedInstanceState.getSparseParcelableArray(VIEWS_TAG);
        if (savedStates != null) {
            mContentParent.restoreHierarchyState(savedStates);
        }
从Bundle中拿出之前保存的SparseArray，丢给顶层View的`restoreHierarchyState`

		public void restoreHierarchyState(SparseArray<Parcelable> container) {
        dispatchRestoreInstanceState(container);
    	}
    	
    	protected void dispatchRestoreInstanceState(SparseArray<Parcelable> container) {
        if (mID != NO_ID) {
        	  //拿到当前View的id对应的Parcelable，注意，这个Parcelable很有可能是之前id相同的View保存的
            Parcelable state = container.get(mID);
            if (state != null) {
                // Log.i("View", "Restoreing #" + Integer.toHexString(mID)
                // + ": " + state);
                mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
                onRestoreInstanceState(state);
                if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                    throw new IllegalStateException(
                            "Derived class did not call super.onRestoreInstanceState()");
                }
            }
        }
    	}
    	
调用到View的`onRestoreInstanceState`，这里让子类去处理拿出的信息。

到这里，可以猜到在ViewGroup的`dispatchRestoreInstanceState`调用super方法(恢复自己)，同时应该会去恢复所有子View，这里不上代码了。

##回到Fragment状态保存的问题

虽然`attach`和`dettach`并不会引起Activity的状态保存，但是由于视图树保存状态的机制可以知道，Fragment也可以只保留自己所持有的View，然后恢复Fragment的视图树。

在`FragmentManager`的`moveToState`方法中，调用了`saveFragmentViewState`

    void saveFragmentViewState(Fragment f) {
        if (f.mView == null) {
            return;
        }
        if (mStateArray == null) {
            mStateArray = new SparseArray<Parcelable>();
        } else {
            mStateArray.clear();
        }
        //f.mView是Fragment的顶层View
        f.mView.saveHierarchyState(mStateArray);
        if (mStateArray.size() > 0) {
            f.mSavedViewState = mStateArray;
            mStateArray = null;
        }
    }
    
 往下走和Activity恢复视图树的过程一样鸟！
 
 这样看起来，ListView能够恢复到上次保存的位置，可以看到AbsListView中的`onSaveInstanceState`和`onRestoreInstanceState`进行了保存和恢复，其中保存一个变量 `mSyncPosition = ss.position;`