---
layout: post
title: "geniousadapter,快速构建Adapter"
date: 2014-12-14 20:28:32 +0800
comments: true
categories: Android
---
项目地址：[**geniousadapter**](https://github.com/chuyun923/ingenious-adapter)<br/>
前面的话：本项目的原型是[QuickAdapter](https://github.com/JoanZapata/base-adapter-helper)，它们的思路基本一致，但本项目的优势在于：

+ 支持AdapterView存在多个layout类型
+ 可配置图片加载缓存库
<!--more-->

在使用AdapterView时，我们需要使用Adapter来绑定数据源和AdapterView中的每一项数据。通常我们继承自BaseAdapter,然后重写四个方法：

    public int getCount()
    
    public Object getItem(int position) 

    public long getItemId(int position)

    public View getView(int position, View convertView, ViewGroup parent) 
    
其中主要的逻辑实现在getView，这个方法主要完成两步操作：1、生成(或者从缓存中取出)当前item对应的ItemView；2、将数据和ItemView绑定。通常，由于AdapterView支持缓存机制(如ListView)，我们通过一个Holder来避免每一次getView重复的findViewById。

	private static class Holder {
		TextView tv_name;
		ImageView iv_avatar;
		.....
	}
	
	public View getView(int position, View convertView, ViewGroup parent) {
		Holder hodler = null;
		if(converView==null) {
			holder = new Holder();
			 convertView = LayoutInflater.from(context)
                .inflate(layoutId, parent, false);
			holder.tv_name = (TextView) findViewById(R.id.tv_name);
			holder.iv_avatar = (ImageView) findViewById(R.id.iv_avatar);
			...
			//下次就不需要findViewById了
			covertView.setTag(holder);
		}
		
		holder =(Holder) convertView.getTag();
		holder.tv_name.setText(***);
		holder.iv_avatar.set(***);
	} 
	
以上就是BaseAdapter的典型用法，那么在项目里面的所有Adapter都存在Holder，并且都存在`holder.properties = (ViewType) findViewById(id)`的重复代码。可以想象一下，如果Holder中有比较多的属性，特别是如果一个AdapterView具有多个不同类型的layout，那么也需要多个不同Holder，getView将会特别复杂。<br />
**geniousadapter**对getView进行了一层封装，并将getView函数的两部分功能进行拆分，自动完成了生成ItemView和Holder的过程，通过一个抽象方法covert让子类实现数据绑定。子类需要实现两个抽象方法：
	  
	  /**
      * 通过AdapterHolder填充view的属性,这个函数主要完成数据绑定的过程，使用方法：
      * holder.setText(R.id.tv_name,"张三").setText(R.id.tv_nickName,
      * "三儿").setImageResource(R.id.iv_avatar,R.drawable.ic_user_avatar);
      * holder 
      * item 当前item需要绑定的数据
      */
      protected abstract void convert(AdapterHolder holder, T item,int viewType);

     /**
      * layoutid至数据类型的映射,插入顺序对应itemviewtype
      * @return
      */
      protected abstract int[] assignLayoutIDs();
      
`holder.setImageUrl(int,imageUrl)`可以通过使用者自己定义远程图片加载的方式。用户可以自己实现加载图片或者使用第三方图片加载缓存库，其接口如下：

	public interface ImageLoader {

    public void load(ImageView imageView,String imageUrl);

    //placeResId  默认图resid
    public void load(ImageView imageView,String imageUrl,int placeResId);
    }
    
比如我们可以使用picasso来完成加载图片的功能，在合适的位置来指定：

	DefaultAdapterConfig.setImageLoader(new ImageLoader() {
		@Override
        public void load(ImageView imageView, String imageUrl) {
            picasso.load(imageUrl).into(imageView);
        }

        @Override
        public void load(ImageView imageView, String imageUrl, int placeResId) {
            picasso.load(imageUrl).placeholder(placeResId).into(imageView);
        }
	});

总结：genious Adapter可以使用户在getView方法中无需关注每一项ItemView生成的细节，而只需要处理数据绑定的逻辑即可。
