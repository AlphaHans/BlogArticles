---
title: 【选择图片Jar】完成选择图片后：预览图片的实现（八）
date: 2016-01-30 13:39 
---
<h1>1.开篇</h1>
在多选模式下，我们选择玩完图片之后，一般都会提供一个图片预览的功能就像微信这样：<!-- more -->
![这里写图片描述](http://img.blog.csdn.net/20160130132141835)
![这里写图片描述](http://img.blog.csdn.net/20160130130447110)

我们来看看我们实现的效果：
![这里写图片描述](http://img.blog.csdn.net/20160130132200617)
![这里写图片描述](http://img.blog.csdn.net/20160130132214168)
![这里写图片描述](http://img.blog.csdn.net/20160130131728713)


细节上有些区别，但功能我们总归是实现了吧！！

<h1>2.开始实现前的准备</h1>
我们先来看看，我们展示**已选择**图片的这个GridView是不是和我们**待选择**图片的是一样的呢？

答案是肯定的！ 所以很明显我们可以继续复用这个控件，也就是V是一样的，但不同的是什么呢？没错是M和C是不一样的，就是Model和Controller是不一样的。

那最复杂一步就是改变Controller的逻辑处理了吧！

<h1>3.抽取BaseController</h1>

我们来分析一下 我们新的Controller需要什么方法：
 
1.获取外部数据源，这个必须有吧？所以我们要有setAdapter

2.控制View的行为，那么setPhotoViewListener这个也必须有。

3.处理View的点击事件，那么click方法也必须有吧？

4.对外暴露，现在预览的图片的路径的集合也要又吧？所以要有个getPaths方法

我们现在看看，这4个方法好像和我们GalleryViewController的方法有点重复啊？

是的，没错！那么我们为了复用这个GalleryGridView以及更加符合软件工程，我们就抽取了一个BaseController!

```
public abstract class BaseController {

	public abstract void setPhotoViewListener(IPhotoGridView view);

	public abstract void setAdapter(List<String> paths);

	public abstract void click(int position);

	public abstract List<String> getPaths();
}

```

然后我们需要修改一下，GalleryGridView里面的Controller的类型，全部从GalleryViewController改为BaseController!

那现在GalleryGridView完整代码就是：

```
public class GalleryGridView extends GridView implements IPhotoGridView,
		android.widget.AdapterView.OnItemClickListener {
	private BaseController mController;

	public GalleryGridView(Context context) {
		this(context, null);
	}

	public GalleryGridView(Context context, AttributeSet attrs) {
		this(context, attrs, 0);
	}

	public GalleryGridView(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
		this.setOnItemClickListener(this);
	}

	public void bindController(BaseController controller) {
		mController = controller;
		mController.setPhotoViewListener(this);
	}

	public void unbindController(BaseController controller) {
		mController.setPhotoViewListener(null);
		mController = null;
	}

	@Override
	public void bindAdapter(BaseAdapter adapter) {
		if (adapter != null)
			this.setAdapter(adapter);
	}

	@Override
	public void onItemClick(AdapterView<?> parent, View view, int position,
			long id) {
		mController.click(position);
	}

	@Override
	public GridView getGridView() {
		return this;
	}
}
```
以及我们原本的GalleryViewController要继承这个BaseController,并且实现里面的方法！

那么原本的GalleryViewController现在的代码就是：

```
public class GalleryViewController extends BaseController {
	protected IPhotoGridView mListener;
	private GalleryAdapter mAdapter;
	/**
	 * Adapter中图片的所有路径
	 */
	private List<String> mPaths;
	/**
	 * 选择图片的最大张数
	 */
	private int mChosenMaxSize;
	/***
	 * 选择了的图片的路径
	 */
	private SparseArray<String> mChosenSpArr;
	/**
	 * 选择图片的计数器
	 */
	private int mChosenCount = 0;
	/**
	 * 选中的图标和未选中的图标
	 */
	private int mSelectedIcon, mUnSelectedIcon;
	/**
	 * 选中和未选中时候的颜色
	 */
	private int mSelectedColor, mUnSelectedColor;

	public static int SINGLE_MODE = 0x123456;

	public static int MUTIPLE_MODE = 0x654321;
	/**
	 * 当前模式
	 */
	private int mModeState;
	/**
	 * 单选模式回调
	 */
	private OnSingleModeClickListener mSingleModeClickListener;

	public GalleryViewController(int mode, int maxSize, int selectedIcon,
			int unselectedIcon) {
		mChosenSpArr = new SparseArray<String>();
		mChosenMaxSize = maxSize;
		mSelectedIcon = selectedIcon;
		mUnSelectedIcon = unselectedIcon;
		mSelectedColor = Color.parseColor("#77000000");
		mUnSelectedColor = Color.parseColor("#00000000");
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

	@Override
	public void setPhotoViewListener(IPhotoGridView listener) {
		mListener = listener;
	}

	@Override
	public void setAdapter(List<String> paths) {
		if (paths == null)
			throw new IllegalArgumentException("paths can not be null");
		if (mListener == null)
			throw new IllegalArgumentException(
					"mListener is null. you should invoke GalleryGridView's bindController()");
		mPaths = paths;
		// mSelectedIcon = selectedIcon;
		// mUnSelectedIcon = unselectedIcon;
		mAdapter = new GalleryAdapter(mListener.getGridView().getContext(),
				paths, mSelectedIcon, mUnSelectedIcon);
		mListener.bindAdapter(mAdapter);
	}

	@Override
	public void click(int position) {
		if (mModeState == SINGLE_MODE) {
			if (mSingleModeClickListener == null) {
				throw new IllegalArgumentException(
						"OnSingleModeClickListener is null,you shuold inovke setSingleModeListener first");
			}
			mSingleModeClickListener.onClick(position, mPaths.get(position));
		} else if (mModeState == MUTIPLE_MODE) {
			// if (position != 0) {
			ImageView selectImageView = (ImageView) mListener.getGridView()
					.findViewWithTag(mPaths.get(position) + "select");

			ImageView showImageView = (ImageView) mListener.getGridView()
					.findViewWithTag(mPaths.get(position));
			// 右上角的选中的脚标管理器
			boolean flag = mAdapter.getSparseBooleanArray().get(position);
			if (selectImageView != null && showImageView != null) {
				if (flag) {
					mAdapter.getSparseBooleanArray().put(position, false);
					// ImageLoader.getInstance().displayImage(
					// "drawable://" + mUnSelectedIcon, selectImageView);
					ImageLoaderWrapper.loadFromDrawable(selectImageView,
							mUnSelectedIcon);
					mChosenCount--;
					mChosenSpArr.remove(position);
					// 选中图层颜色变换
					showImageView.setColorFilter(mUnSelectedColor);
				} else {
					if (mChosenCount == mChosenMaxSize) {
						Toast.makeText(mListener.getGridView().getContext(),
								"只能选择" + mChosenMaxSize + "张图片",
								Toast.LENGTH_SHORT).show();
						return;
					}
					showImageView.setColorFilter(mSelectedColor);
					mAdapter.getSparseBooleanArray().put(position, true);
					// ImageLoader.getInstance().displayImage(
					// "drawable://" + mSelectedIcon, selectImageView);
					ImageLoaderWrapper.loadFromDrawable(selectImageView,
							mSelectedIcon);
					mChosenCount++;
					mChosenSpArr.put(position, mPaths.get(position));
				}
			}
		}
	}

	@Override
	public List<String> getPaths() {
		List<String> chosenPhotoList = new ArrayList<String>();
		// 遍历SpareArray数组，这里的有点意思
		int key;
		// System.out.println("mChosenSpArr size = " + mChosenSpArr.size());
		for (int i = 0; i < mChosenSpArr.size(); i++) {
			key = mChosenSpArr.keyAt(i);
			String value = mChosenSpArr.get(key);
			// System.out.println("key = " + key + " value = " + value);
			chosenPhotoList.add(value);
		}
		return chosenPhotoList;
	}

	public void update(List<String> paths) {
		if (paths == null)
			throw new IllegalArgumentException("paths can not be null");
		if (mAdapter == null)
			throw new IllegalArgumentException(
					"adapter can not be null,you should invoke setAdapter() first,which will create PhotoAdapter's Instance");
		mPaths = paths;
		mChosenSpArr.clear();// 已选择的清理
		mAdapter.update(mPaths);
	}

	public void setSingleModeListener(OnSingleModeClickListener listener) {
		mSingleModeClickListener = listener;
	}

	public interface OnSingleModeClickListener {
		void onClick(int position, String path);
	}
}
```

OK！现在我们只需要创建多一个新的Controller来实现符合我们预期的逻辑！ 

我们创建一个类：叫ChosenPhotoViewController 

并且继承BaseController!

那么现在的代码就长这样了：

```
public class ChosenPhotoViewController  extends BaseController {

	@Override
	public void setPhotoViewListener(IPhotoGridView view) {

	}

	@Override
	public void setAdapter(List<String> paths) {

	}

	@Override
	public void click(int position) {

	}

	@Override
	public List<String> getPaths() {
		return null;
	}

}
```
OK！开发的前期准备工作我们就做到这里！ 一个BaseController就这样诞生了！

<h1>4.结束语</h1>
这里我们抽取了一个BaseController，我怕还有的同学没有反应过来，所以没有看懂的没有关系，多看几遍，好好消化一下。

我们下一篇课程将带来具体逻辑的实现~所以没有搞懂的请务必要在这里看明白了再继续看下去！