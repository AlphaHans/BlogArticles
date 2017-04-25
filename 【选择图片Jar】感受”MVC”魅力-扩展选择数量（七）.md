---
title: 【选择图片Jar】感受”MVC”魅力-扩展选择数量（七）
date: 2016-01-29 18:33
---
<h3>1.前言</h3>
上篇文章是说会带来本Jar的第一次扩展。本次扩展的是规定选择数量上限！

需求分析，我们选择图片无外乎两种情况：
第一种：选择多张图片，作为附件上传；
第二种：选择一张图片，作为头像，然后再上传。
<!-- more -->
因此衍生出我们扩展的需求：
1.多选模式 设置选择图片数量上限
2.单选模式 选择图片完成后 然后回调处理

这个时候，MVC的魅力就来了。我们仅需要在C中改少量代码，即可完成我们的需求！

接下来我们一步一步分析，我们修改了什么代码，或者添加了什么！

<h3>2.新的构造方法</h3>

```
	/**
	 * 单选模式
	 */
	public static int SINGLE_MODE = 0x123456;
	/**
	 * 多选模式
	 */
	public static int MUTIPLE_MODE = 0x654321;
	/**
	 * 当前模式
	 */
	private int mModeState;
	/**
	 * 选择图片的最大张数
	 */
	private int mChosenMaxSize;
	/**
	 * 选择图片的计数器
	 */
	private int mChosenCount = 0;
	
	public GalleryViewController(int mode, int maxSize) {
		mChosenSpArr = new SparseArray<String>();
		//---新增以下代码---//
		mChosenMaxSize = maxSize;
		// 如果不是这两种模式的一种，默认为多选模式
		if (mode != SINGLE_MODE || mode != MUTIPLE_MODE) {
			mModeState = MUTIPLE_MODE;
		} else {
			mModeState = mode;
		}
		// 如果是单选模式 强制只能选择一张
		if (mode == SINGLE_MODE) {
			mChosenMaxSize = 1;
		}
	}
```

OK！我们添加了几个必要的参数作为，并且构造方法新增了一些代码！ 也不是很多而已。相信大家都能看懂。

<h3>3.单选模式</h3>
经过上面的构造方法，规定了是单选模式还是多选模式。我们现在来想想，还需要添加什么才能够完成我们的需求。

在上面的分析写到，如果是单选，选择则直接回调接口。

因此我们还需要对外暴露一个回调接口。（PS:如果新手对回调思想不理解，请查阅相关知识。 或者尝试理解以下代码）

所以，我们需要添加以下代码：

```
	/**
	 * 单选模式回调
	 */
	private OnSingleModeClickListener mSingleModeClickListener;
	
	public void setSingleModeListener(OnSingleModeClickListener listener) {
		mSingleModeClickListener = listener;
	}

	public interface OnSingleModeClickListener {
		void onClick(int position);
	}
```
然后我们还需要处理哪里呢？

没错，就是click方法里面的判断，添加以下代码：

```
public void click(int position) {
		// 如果是单选模式
		if (mModeState == SINGLE_MODE) {
			if (mSingleModeClickListener == null) {
				throw new IllegalArgumentException(
						"OnSingleModeClickListener is null,you shuold inovke setSingleModeListener first");
			}
			mSingleModeClickListener.onClick(position);
			
			//多选模式
		} else if (mModeState == MUTIPLE_MODE) {
			...//原来的代码
				}
			}
		}
	}

``` 

你看 我们只需要在这里加上这一点代码就OK了！
然后我们使用起来也非常简单：

```
mController = new GalleryViewController(
				GalleryViewController.SINGLE_MODE, 1);
mController.setSingleModeListener(new 
		OnSingleModeClickListener() {
			@Override
			public void onClick(int position) {
				// 逻辑处理~
			}
		});
```

当然啦，这里可以不单单穿position，也可以带上这个图片的路径，那回调接口就要改成：

```
public interface OnSingleModeClickListener {
	void onClick(int position, String path);
}
```

click代码就要改成：


```
mSingleModeClickListener.onClick(position, mPaths.get(position));
```

所以要什么都可以自己加！~

<h3>4.多选模式</h3>
现在我们来想想多选模式还缺少什么？ 好像并不需要像单选加那么多东西了。我们仅需要判断一下当前数量数和最大数量就可以了！！

```
if (flag) {
	...
	mChosenCount--;
} else {
	if (mChosenCount == mChosenMaxSize) {
		//弹出一个Toast提示只能选择mChosenMaxSize张图片
		return;
	}
	...
	mChosenCount++;
}
```

没错 多选模式就只要加这么一点点！ 是不是出乎自己的意料呢！所以一个合理的开发模式，是高效开发和扩展的保障！

<h3>5.结束语</h3>
至此，一个选择图片已经完成80%了！是不是有点小激动呢！ 那么还有20%呢？ 

还记得这只是第一次扩展吧？ 接下来的第二次扩展，将是为了业务逻辑而扩展，更加符合我们实际需求。 

卖个关子先~~ 红红火火恍恍惚惚

OK！这篇文章就到这里！