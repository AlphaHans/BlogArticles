---
title: 【选择图片Jar】Adapter的item布局的实现（三）
date: 2016-01-29 14:56
---
<h3>1.回到原点</h3>
还记得我在第一篇文章中提及要用代码来写布局，来摆脱jar不能打包资源文件的困扰。 今天就是要带来如何用代码写布局文件。 
<!-- more -->
<h3>2.布局分析</h3>
首先我们看看我们要实现的布局是什么样子的。截取了一个item
![这里写图片描述](http://img.blog.csdn.net/20160129143155915)

我们可以看到，实际上是两个ImageView组成，我相信大家用布局来写，不超过一分钟就写完了。但是用代码实现可能就要下一番功夫了~

<h3>3.布局实现</h3>

 1. 首先编写一个类 继承自RelativeLayout
 2. 重写他的构造方法

```
public class ChosePhotoItemLayout extends RelativeLayout {
	private ImageView mShow;
	private ImageView mSelect;

	public ChosePhotoItemLayout(Context context) {
		this(context, null);
	}

	public ChosePhotoItemLayout(Context context, AttributeSet attrs) {
		this(context, attrs, 0);
	}

	public ChosePhotoItemLayout(Context context, AttributeSet attrs,
			int defStyle) {
		super(context, attrs, defStyle);
		this.setLayoutParams(new RelativeLayout.LayoutParams(
				LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
		initContentView();
	}
}
```

 3.Ok，我们看到里面有一个initContentView

```
// CotentView 中只有 初始化两个ImageView
	private void initContentView() {
		initShowImgaeView();
		initSelectImageView();
	}

```
  
  4.先来看看initShowImageView()这个方法：
  这个初始化的是那个惊讶大猫的ImageView

```
	private void initShowImgaeView() {
		mShow = new ImageView(getContext());
		// 图片居中
		mShow.setScaleType(ScaleType.CENTER_CROP);
		// 获取屏幕宽度
		int screenWidth = ScreenUtil.getScreenWidth(getContext());
		// 取屏幕宽度1/3来作为ImageView的大小
		int showViewWidth = screenWidth / 3;
		// 设置ImageView参数
		mShow.setLayoutParams(new ViewGroup.LayoutParams(showViewWidth,
				showViewWidth));
	    // 添加到RelativeLayout父布局中去
		this.addView(mShow);
	}

```

   相信这段代码也非常好理解。ScreenUtil这个是我封装好的一个工具类，具体代码在最后贴出
  
   5.依葫芦画瓢 我们看看initSelectImageView这个方法
   

```
	private void initSelectImageView() {
				mSelect = new ImageView(getContext());
		// 图片居中显示
		mSelect.setScaleType(ScaleType.CENTER_CROP);
		// 默认处理
		int screenWidth = ScreenUtil.getScreenWidth(getContext());
		int selectViewWidth = screenWidth / 18;

		RelativeLayout.LayoutParams p = new LayoutParams(selectViewWidth,
				selectViewWidth);
		// 位于父布局右侧
		p.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
		// 位于父布局上侧
		p.addRule(RelativeLayout.ALIGN_PARENT_TOP);
		// 距离
		int margin = selectViewWidth / 4;
		// 右边距离
		p.rightMargin = margin;
		// 左边距离
		p.topMargin = margin;
		this.addView(mSelect, p);
	}
```
相信这段代码也非常好理解。 需要注意的是RelativeLayout.LayoutParams 一定要注意导正确的包，一开始我用了ViewGroup.LayoutParams折腾了很久都 没有成功将箭头成功放在布局的右上角！！

至此布局文件的核心已经写好了~~！！附上完整代码以及ScreenUtil的代码。

```
package com.hans.selectphoto.view.itemlayout;

import com.hans.selectphoto.util.ScreenUtil;
import com.nostra13.universalimageloader.utils.L;

import android.content.Context;
import android.util.AttributeSet;
import android.view.ViewGroup;
import android.widget.ImageView;
import android.widget.ImageView.ScaleType;
import android.widget.RelativeLayout;

public class ChosePhotoItemLayout extends RelativeLayout {
	private ImageView mShow;
	private ImageView mSelect;

	public ChosePhotoItemLayout(Context context) {
		this(context, null);
	}

	public ChosePhotoItemLayout(Context context, AttributeSet attrs) {
		this(context, attrs, 0);
	}

	public ChosePhotoItemLayout(Context context, AttributeSet attrs,
			int defStyle) {
		super(context, attrs, defStyle);
		this.setLayoutParams(new RelativeLayout.LayoutParams(
				LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
		initContentView();
	}

	// CotentView 中只有 初始化两个ImageView
	private void initContentView() {
		initShowImgaeView();
		initSelectImageView();
	}

	private void initSelectImageView() {
		mSelect = new ImageView(getContext());
		// 图片居中显示
		mSelect.setScaleType(ScaleType.CENTER_CROP);
		// 默认处理
		int screenWidth = ScreenUtil.getScreenWidth(getContext());
		int selectViewWidth = screenWidth / 18;

		RelativeLayout.LayoutParams p = new LayoutParams(selectViewWidth,
				selectViewWidth);
		// 位于父布局右侧
		p.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
		// 位于父布局上侧
		p.addRule(RelativeLayout.ALIGN_PARENT_TOP);
		// 距离
		int margin = selectViewWidth / 4;
		// 右边距离
		p.rightMargin = margin;
		// 左边距离
		p.topMargin = margin;
		this.addView(mSelect, p);
	}

	private void initShowImgaeView() {
		mShow = new ImageView(getContext());
		// 图片居中
		mShow.setScaleType(ScaleType.CENTER_CROP);
		// 获取屏幕宽度
		int screenWidth = ScreenUtil.getScreenWidth(getContext());
		// 取屏幕宽度1/3来作为ImageView的大小
		int showViewWidth = screenWidth / 3;
		// 设置ImageView参数
		mShow.setLayoutParams(new ViewGroup.LayoutParams(showViewWidth,
				showViewWidth));
		// 添加到RelativeLayout父布局中去
		this.addView(mShow);
	}

	public ImageView getShowImageView() {
		return mShow;
	}

	public ImageView getSelectImageView() {
		return mSelect;
	}
}

```
ScreenUtil的代码：

```
package com.hans.selectphoto.util;

public class ScreenUtil {

	public static int getScreenWidth(Context context) {
		WindowManager wm = (WindowManager) context
				.getSystemService(Context.WINDOW_SERVICE);
		DisplayMetrics outMetrics = new DisplayMetrics();
		wm.getDefaultDisplay().getMetrics(outMetrics);
		return outMetrics.widthPixels;
	}

	public static int getScreenHeight(Context context) {
		WindowManager wm = (WindowManager) context
				.getSystemService(Context.WINDOW_SERVICE);
		DisplayMetrics outMetrics = new DisplayMetrics();
		wm.getDefaultDisplay().getMetrics(outMetrics);
		return outMetrics.heightPixels;
	}
}

```
好的！至此，我们成功用代码实现布局啦~~~~~下面我们就看看如何使用了~
<h3>4.使用布局</h3>
首先我们来回顾一下我们原来的是如何使用layout下的布局的？只需要：

```
convertView = LayoutInflater.from(context).inflate(R.id.xx,....);
```
如何获取布局文件中的控件呢？只需要：

```
viewHolder.show = (ImageView) 
	convertView .findViewById(R.id.grid_iv_pic);
viewHolder.select = (ImageView)
	convertView.findViewById(R.id.grid_iv_button);
```

就可以成功使用了吧？ 那现在我们要如何使用呢？嘿嘿。。

其实只需要new就好了- -#没错就是这么简单 - -#是不是碉堡了。 哈，开个玩笑。

```
convertView = new ChosePhotoItemLayout(mContext);
```

那我们获取控件呢？还记得我们对外暴露了两个方法吧？只需要：

```
viewHolder.show = ((ChosePhotoItemLayout) convertView)
					.getShowImageView();
viewHolder.select = ((ChosePhotoItemLayout) convertView)
					.getSelectImageView();
```

<h3>5.结束语</h3>
ok! 今天的文章也完毕啦~应该都学会了吧。 编程这种东西还是要自己打过才知道。有很多时候就是因为导错包而导致了解决不了问题。比如我的SelectImageView的那个RelativeLayout.LayoutParams的问题！

好的Adapter的所有的代码也慢慢的浮出水面啦~下一篇文章将带着大家揭开GalleryAdapter神秘面纱！！！ 其实一点都不神秘- -#