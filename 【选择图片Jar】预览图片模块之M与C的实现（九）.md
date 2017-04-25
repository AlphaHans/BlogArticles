---
title: 【选择图片Jar】预览图片模块之M与C的实现（九）
date: 2016-01-30 14:44 
---
<h1>1.前言</h1>
经过上一篇文章的铺垫，如果已经完全搞懂了，那么我们就可以开始进行我们本节的内容了！

本篇文章将带来，完整的Model的代码，以及对绝大部分的Controller的代码。 

<!-- more -->

<h1>2.ChosenPhotoAdapter的实现</h1>

依旧，我们从构造方法开始看：

```
	/**
	 * 上下文
	 */
	private Context mContext;
	/**
	 * 图片路径
	 */
	private List<String> mPaths;
	/**
	 * 每行图片数量 -用于计算ImageView的大小
	 */
	private int mNumColumns;
	/**
	 * 添加图片的资源id
	 */
	private int mAddIconId;
	
	public ChosenPhotoAdapter(Context context, List<String> paths,int numColumns, int addIconId) {
		mContext = context;
		mNumColumns = numColumns;//每行图片显示的数量
		mPaths = new ArrayList<String>();
		mPaths.addAll(paths);
		//标记最后一位！
		mPaths.add("");
		mAddIconId = addIconId;//+号图片的Id
	}
```
相信大家看了注释应该都能知道各个参数的意思。

需要特别注意的是 mNumColumns这个，这个我们主要是用来在getView()计算 ImageView宽高的！

然后我们在mPaths加入一个“”作为标记位。

接下来我们来看看核心的方法：getView()的代码：

```
if (convertView == null) {
	int screenWidth = ScreenUtil.getScreenWidth(mContext);
	//int screenHeight =ScreenUtil.getScreenHeight(mContext);
	
	// 计算图片的宽度 -目的是用来作为图片的高，保持图片方正，比较好看
    int cWidth = screenWidth / mNumColumns;
	
	ImageView show = new ImageView(mContext);
	show.setLayoutParams(new ViewGroup.LayoutParams(
			android.view.ViewGroup.LayoutParams.MATCH_PARENT, cWidth));
			
	show.setScaleType(ScaleType.CENTER_CROP);
} 
// 判断是不是最后一位，如果是最后一位则加载资源图片，不是路径
if (position != mPaths.size() - 1) {
	ImageLoaderWrapper.loadFromFile((ImageView) convertView,mPaths.get(position));
} else {
	ImageLoaderWrapper.loadFromDrawable((ImageView) convertView,mAddIconId);

}
return convertView;
```
相信这段代码也没有什么难理解的地方。 

不过我稍微提及一下这里。

```
if (position != mPaths.size() - 1) 
```
这个判断就是判断是否到了标记位的位置！

来看看最后一个方法update()

```
	public void update(List<String> list) {
		if (mPaths == null) {
			mPaths = new ArrayList<String>();
		}
		mPaths.clear();
		mPaths.addAll(list);
		// 标记最后一位
		mPaths.add("");
		notifyDataSetChanged();
	}
```
这里也是要处理一个标记位相信大家都能理解。

OK！那么Model的改造就到这里了！如果前面的文章都搞懂了，那么相信这里的逻辑也很清晰明了了！

接下来就看看Controller了！

<h1>3.ChosenPhotoViewController的实现</h1>
依旧从构造方法看起！

```
	/**
	 * 一行图片的数量
	 */
	private int mNumColumuns;
	/**
	 * +号图片的资源Id
	 */
	private int mAddIconId;
	}
	/**
	 * 图片路径
	 */
	private List<String> mPaths;
	
	private IPhotoGridView mView;
	
	private ChosenPhotoAdapter mAdapter;
	
	public ChosenPhotoViewController(int numColumns, int addIconId) {
		mAddIconId = addIconId;
		mNumColumuns = numColumns;
	}
```
相信很好理解。

我们来着重看看这个方法：

```
@Override
public void setPhotoViewListener(IPhotoGridView view) {
	mView = view;
	mAdapter = new 
	ChosenPhotoAdapter(mView.getGridView().getContext(),
				mPaths, mNumColumuns, mAddIconId);
	mView.bindAdapter(mAdapter);
}
```
我们知道，当我们的View调用了bindController之后，会自动调用对应Controller的setPhotoViewListener  代码如下：

```
public class GalleryGridView extends GridView implements IPhotoGridView,
		android.widget.AdapterView.OnItemClickListener {
	private BaseController mController;

	.......

	public void bindController(BaseController controller) {
		mController = controller;
		mController.setPhotoViewListener(this);
	}
	
	.......
}
```

那我们为何要在setPhotoViewListener里面创建Adapter呢？而不是在setAdapter的方法才创建呢？ 

**这是因为，我们要默认显示出一个+号的图片啊！ 所以我们要先创建Adapter!!**  

就是这个：![这里写图片描述](http://img.blog.csdn.net/20160130142508993)

搞明白了这个就迈过了我们的第一道坎了！  那么我们现在来看看setAdapter的代码就很好理解啦！

```
@Override
public void setAdapter(List<String> paths) {
	if (mView == null)
		throw new IllegalArgumentException(
					"mListener is null. you should invoke ChosenPhotoGridView's bindController() first");
	mPaths = paths;
	mAdapter.update(mPaths);
}
```
那还剩下一个最困难的方法了！ 我们需要单独拿出来好好讲一下！

<h1>4.Click中的逻辑处理</h1>
我们来看看click逻辑处理的框架长什么样子：

```
	@Override
	public void click(int position) {
		//注意这个是 mPaths.size() 不是 mPaths.size()-1
		if (position == mPaths.size()) {
			//跳转选择图片页面
		} else {
			//显示详情页面
		}
	}
```
需要单独把这段代码拿出来讲讲：

```
if (position == mPaths.size())
```
还记得我们在Adapter中设置多了一个“”的标记位吧？那这个标记位是第几个位置呢？ 这个需要思考一下哦，反应过来了就知道我这里判断的意思了！

那接下来我们来实现具体的逻辑了。

首先是+号图片的点击事件的处理，也就是这段代码：

```
if (position == mPaths.size()) {
	//跳转选择图片页面
	//我们要在这里写.....
}else{
	
}
```
首先我们这里要进行页面的跳转，也就是要跳转到选择图片的那个Activity。

是我们这个是Jar包啊！ 不能够知道具体项目中要跳转到的目标的Activity，那我们怎知道呢？

答案是：我们不知道！ 哈哈~那我们怎么办？相信有的同学已经知道了，我们可以接口回调啊！具体的逻辑让使用Jar的人来处理啊！

没错了 就是这个！

那代码就变成了：

```
	/**
	 * +号图片，接口回调
	 */
	private AddIconAction mAction;
	
	@Override
	public void click(int position) {
		if (position == mPaths.size()) {
			//跳转选择图片页面 逻辑用接口回调让外部处理！
			if (mAction == null) {
				throw new IllegalArgumentException(
						"AddIconAction is null. you should invoke setAddIconAction first");
			}
			mAction.onAddClick();
		} else {
			//跳转到详情页面
		}
	}

	public void setAddIconAction(AddIconAction action) {
		mAction = action;
	}

	public interface AddIconAction {
		void onAddClick();
	}

```

这样只是不是就比较好地处理了这个尴尬的事件了呢？

那么我们现在来看看**“跳转到详情页面“**这个逻辑。 有的同学可能就说了，我知道，这里还是用接口回调，因为我们需要跳转到另外一个Activity去展示我们的图片！

诚然，这是一个方法，但未必是一个好方法。 因为我们想想，我们这是一个**Jar包，是为了抽取公用模块，达到快速开发的目的。**那如果再次使用接口回调，跳转页面，岂不是我们又要编写多一个新的Activity?这个Activity要包括我们上面的Bar还有 ViewPager，然后还要编写一大堆的逻辑处理代码。NONONO，**这不是我们所需要的！**

**那怎么办？**

<h1>5.结束语</h1>
我们留一个悬念，我们到底要如何解决这个问题？希望都可以开动一下自己的脑袋瓜，思考一下！

我一直认为”授人以鱼，不如授人以渔“。唯有我们开动脑经，才能发现更多的实现方法！

这个悬念将在下篇文章揭晓！ 如果有答案的同学，也可以在下面评论踊跃发表一下自己的见解！
