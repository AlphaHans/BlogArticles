---
title: 【选择图片Jar】”MVC“核心Controller的代码分析（六）
date: 2016-01-29 17:06
---
<h3>1.开篇</h3>
红红火火恍恍惚惚，终于来到重点啦！GalleryViewController的完整代码实现！

各位小伙伴，上好厕所拿好板凳开始啦。这里信息量有点大！
<!-- more -->
<h3>2.回顾</h3>
我们从最开始的构造方法讲起！

还记得我们Controller的职责吗？ 负责M和V的逻辑通信！
也就是C要负责拿到原始数据->构建M->通过C->设置给V！

我们回顾一下V如何和C建立联系的？通过bindController对吧！

 然后C如何设置V呢？ 通过IPhotoGridView接口中的bindAdapter！

梳理清楚里之后，我们开始看C的代码吧！！

<h3>3.构造方法</h3>

```
	/***
	 * 选择了的图片的路径
	 */
	private SparseArray<String> mChosenSpArr;
	
	public GalleryViewController() {
			mChosenSpArr = new SparseArray<String>();
	}
```
我们创建了一个SparseArray，用来保存选择的路径。

有的同学可能有疑问了。为何不用List来保存？ 这个是个好问题！

还记得我们将V的点击事件传送到了C来做逻辑处理吧？

那考虑到用户可能点击了这个Item,当他重复点击多一次，是不是说明用户不选择这张图片了？那我们是不是要删除已经保存了的这个选择图片的路径！ 

所以我们用SparseArray来保存！如果看不懂我在讲什么的。请回顾这一篇文章的相应段落。[【选择图片Jar】Adapter的实现之SparseBooleanArray图片状态管理器（二）](http://blog.csdn.net/qq_18402085/article/details/50606902)

<h3>4.核心方法之setAdapter()</h3>

```
	public void setAdapter(List<String> paths, int selectedIcon,
			int unselectedIcon) {
		if (paths == null)
			throw new IllegalArgumentException("paths can not be null");
		if (mListener == null)
			throw new IllegalArgumentException(
					"mListener is null. you should invoke GalleryGridView's bindController()");
		// 外部传入的图片路径			
		mPaths = paths;
        // 选中和未选中状态的图片
		mSelectedIcon = selectedIcon;
		mUnSelectedIcon = unselectedIcon;
		// 创建Adapter
		mAdapter = new GalleryAdapter
		(mListener.getGridView().getContext(),
				paths, selectedIcon, unselectedIcon);
		//让View绑定Adapter
		mListener.bindAdapter(mAdapter);
	}
```

这段代码也很好理解。

首先我们对传入的参数进行了一些必要的判断。

然后创建了Adpater。

然后C通过接口对V设置了数据源。

<h3>5.核心方法之click</h3>
这个方法名字还记得吗。 是重写了V的onItemClick讲点击事件的逻辑传送到C的方法！

```
@Override
public void onItemClick(AdapterView<?> parent, View view, int position,long id) {
	mController.click(position);
}

```

OK！ 想起来这个方法了。那我们就来具体分析一下C是如何处理这个点击事件的逻辑的！

```
public void click(int position) {
	ImageView selectImageView = (ImageView) mListener.getGridView().findViewWithTag(mPaths.get(position) + "select");

	ImageView showImageView = (ImageView) mListener.getGridView().findViewWithTag(mPaths.get(position));
	// 右上角的选中的脚标管理器
	boolean flag = 
			mAdapter.getSparseBooleanArray().get(position);
   
   if (selectImageView != null && showImageView != null) {
	
	if (flag) {
	mAdapter.getSparseBooleanArray().put(position, false);

	ImageLoaderWrapper.loadFromDrawable(selectImageView,
							mUnSelectedIcon);
	
    mChosenSpArr.remove(position);
	
	// 选中图层颜色变换
	showImageView.setColorFilter(mUnSelectedColor);
	
	} else {
		
		showImageView.setColorFilter(mSelectedColor);
		
		mAdapter.getSparseBooleanArray().put(position, true);
				
		ImageLoaderWrapper.loadFromDrawable(selectImageView,
							mSelectedIcon);
	    
        mChosenSpArr.put(position, mPaths.get(position));
		}
	}
}
```

OK！完整代码如上，我们一段一段来分析！


```
ImageView selectImageView = (ImageView) mListener.getGridView().findViewWithTag(mPaths.get(position) + "select");

ImageView showImageView = (ImageView) mListener.getGridView().findViewWithTag(mPaths.get(position));
```

还记得我们在 
[【选择图片Jar】拨云见日-GalleryAdapter完整代码（四）](http://blog.csdn.net/qq_18402085/article/details/50607597) 
的GalleryAdapter中的getView方法为两个ImageView的setTag方法吗？

这里通过getGridView()返回的View实例，调用findViewWithTag找到我们的ImageView。


```
boolean flag = mAdapter.getSparseBooleanArray().get(position);
if (selectImageView != null && showImageView != null) {
	....
}
```

获取当前位置是否已经被保存过，即是否已经被点击！

然后判断一下我们是否通过findViewWithTag找到我们的ImageView!

```

if (flag) {
		//修改选择的状态
		mAdapter.getSparseBooleanArray().put(position, false);
		//修改右上角脚标图片								 
		ImageLoaderWrapper.loadFromDrawable(selectImageView,
							mUnSelectedIcon);
		//移出已经保存的路径
		mChosenSpArr.remove(position);
									 
		// 选中图层颜色变换		
		showImageView.setColorFilter(mUnSelectedColor);

}
```

这里的逻辑看到注释也应该很好理解了！

那else里面的逻辑代码，也就是相反而已！

<h3>6.对外暴露已经选择图片的方法</h3>

```
	public List<String> getChosenList() {
		List<String> chosenPhotoList = new ArrayList<String>();
		// 遍历SpareArray数组，这里的有点意思
		int key;
		for (int i = 0; i < mChosenSpArr.size(); i++) {
			key = mChosenSpArr.keyAt(i);
			String value = mChosenSpArr.get(key);
			chosenPhotoList.add(value);
		}
		return chosenPhotoList;
	}
	
```

对外返回一个List集合，更加符合我们日常的使用习惯！

<h3>7.结束语</h3>
至此，一个封装好的jar已经基本搞定了！

为什么说是基本呢？

我们想想看。一般我们选择图片都是用来附件上传或者作为头像的。参照微信来说，最大照片数量是9张，头像就很明显是1张。那我们是不是要针对多选和单选进行不同的逻辑处理呢！

答案是肯定的！ 嘿嘿这个时候MVC开发模式的优势就出来了！我们完全不需要修改M和V的代码，我们仅仅需要对C进行适当的修改，就能完美做到我们的要求啦！是不是想象都激动！

OK下篇文章讲带来这个基本jar包的第一次扩展！为什么是第一次呢，先卖个关子，讲完下篇文章之后，后续会带来第二次扩展！
