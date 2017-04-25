---
title: 【选择图片Jar】拨云见日-GalleryAdapter完整代码（四）
date: 2016-01-29 15:34
---
<h3>1.开篇</h3>
经过前面（二）（三）两篇文章的基础，现在终于迎来激动人心的时刻啦！！拨云见日-GalleryAdapter的完整代码！
<!-- more -->
<h3>2.构造方法</h3>
首先我们看看他的构造方法和成员变量

```
public class GalleryAdapter extends BaseAdapter {
	// 路径集合
	private List<String> mPaths;
	// 上下文
	private Context mContext;
	// 选中状态管理器
	private SparseBooleanArray mSparseManager;
	// 右上角脚标的图片id
	private int mSelectedIcon, mUnSelectedIcon;
	// 选中状态的ColorFilter颜色代码
	private int mSelectedColor, mUnSelectedColor;

	public GalleryAdapter(Context context, List<String> list, int selectedIcon,
			int unselectedIcon) {
		this.mPaths = list;
		this.mContext = context;
		mSparseManager = new SparseBooleanArray();
		mSelectedIcon = selectedIcon;
		mUnSelectedIcon = unselectedIcon;
		mSelectedColor = Color.parseColor("#77000000");
		mUnSelectedColor = Color.parseColor("#00000000");
	}
}
```
需要解释的是这两个mSelectedIcon 和mUnSelectedIcon。在第一篇文章说了，我们不能够将资源文件打包进入jar,所以这里需要让项目传入两个id，一个选中状态图片的id一个是未选中状态图片的id。
具体指的是，右上角的那两个图标~
1.选中状态：
![这里写图片描述](http://img.blog.csdn.net/20160129151112311)
2.未选中状态：
![这里写图片描述](http://img.blog.csdn.net/20160129151147806)

<h3>3.核心方法getView()</h3>
像继承于BaseAdapter的另外三个方法就不说啦，我们来看看核心的getView()方法中的处理。代码！代码！我们来看看代码！

```
@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		ViewHolder viewHolder = null;
		if (convertView == null) {
			viewHolder = new ViewHolder();
			convertView = new ChosePhotoItemLayout(mContext);
			viewHolder.show =((ChosePhotoItemLayout)convertView)
							.getShowImageView();
			viewHolder.select = ((ChosePhotoItemLayout) convertView).getSelectImageView();
			
			convertView.setTag(viewHolder);
		} else {
			viewHolder = (ViewHolder) convertView.getTag();
		}
		//当前图片的path
		String path = mPaths.get(position);
		
		//设置ShowImageView的tag
		viewHolder.show.setTag(path);
		ImageLoaderWrapper.loadFromFile(viewHolder.show, path);
		//设置SelectImageView的tag
		viewHolder.select.setTag(path + "select");
		// 处理滑动错位
		handleScrollBackgroundChange(position, viewHolder.show,
				viewHolder.select);
		
		return convertView;
	}
``` 
大体上没有什么特别的。还是老套路创建一个ViewHolder然后进行控件缓存等等。

```
convertView = new ChosePhotoItemLayout(mContext);

viewHolder.show =((ChosePhotoItemLayout)convertView)
				.getShowImageView();
				
viewHolder.select = ((ChosePhotoItemLayout)convertView).getSelectImageView();
``` 
这段代码看不懂的 可以回顾一下上一篇文章[【选择图片Jar】Adapter的item布局的实现（三）](http://blog.csdn.net/qq_18402085/article/details/50607279)

OK！下面我们来看看这次比较重要的两段代码：

```
//设置ShowImageView的tag
viewHolder.show.setTag(path);
```
以及
```
//设置SelectImageView的tag
viewHolder.select.setTag(path + "select");
```
这两段代码为后面的点击事件做铺垫的~如果不明白这个是什么作用的可以理解成和findViewById一样，我们为每个控件做了唯一的标识。

方便我们处理GridView的点击事件~然后更改相应的点击状态。
```
// 处理滑动错位
handleScrollBackgroundChange(position, viewHolder.show,
				viewHolder.select);
```
这段代码看不懂的可以回顾一下第二篇文章[【选择图片Jar】Adapter的实现之SparseBooleanArray图片状态管理器（二）](http://blog.csdn.net/qq_18402085/article/details/50606902)

OK!至此 核心的方法getView()已经分析完毕

<h3>4.其他两个方法</3>

```
    /**
    * 对外暴露图片管理器
    */
	public SparseBooleanArray getSparseBooleanArray() {
		return mSparseManager;
	}
	
	/**
	* 更新Adpter数据的方法
	*/
    public void update(List<String> list) {
		mPaths = list;
		mSparseManager.clear();//特别注意清除所有保存的状态
		this.notifyDataSetChanged();
	}
```
相信这两个方法也非常好理解~！！

<h3>5.GalleryAdapter完整代码</h3>
OK!经过几篇文章的铺垫的描述，相信GalleryAdapter完整代码已经在各位的心中浮出水面了~ 下面贴上完整代码

```
public class GalleryAdapter extends BaseAdapter {
	// 路径集合
	private List<String> mPaths;
	// 上下文
	private Context mContext;
	// 选中状态管理器
	private SparseBooleanArray mSparseManager;
	// 右上角脚标的图片id
	private int mSelectedIcon, mUnSelectedIcon;
	// 选中状态的ColorFilter颜色代码
	private int mSelectedColor, mUnSelectedColor;

	public GalleryAdapter(Context context, List<String> list, int selectedIcon,
			int unselectedIcon) {
		this.mPaths = list;
		this.mContext = context;
		mSparseManager = new SparseBooleanArray();
		mSelectedIcon = selectedIcon;
		mUnSelectedIcon = unselectedIcon;
		mSelectedColor = Color.parseColor("#77000000");
		mUnSelectedColor = Color.parseColor("#00000000");
	}

	@Override
	public int getCount() {
		return mPaths.size();
	}

	@Override
	public Object getItem(int position) {
		return mPaths.get(position);
	}

	@Override
	public long getItemId(int position) {
		return position;
	}

	@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		ViewHolder viewHolder = null;
		if (convertView == null) {
			viewHolder = new ViewHolder();
			// convertView = mLayoutInflater.inflate(R.layout.item_grid, parent,
			// false);
			// viewHolder.show = (ImageView) convertView
			// .findViewById(R.id.grid_iv_pic);
			// viewHolder.select = (ImageView) convertView
			// .findViewById(R.id.grid_iv_button);
			convertView = new ChosePhotoItemLayout(mContext);
			viewHolder.show = ((ChosePhotoItemLayout) convertView)
					.getShowImageView();
			viewHolder.select = ((ChosePhotoItemLayout) convertView)
					.getSelectImageView();
			convertView.setTag(viewHolder);

		} else {
			viewHolder = (ViewHolder) convertView.getTag();
		}
		// if (position == 0) {
		// ImageLoader.getInstance().displayImage("drawable://" + mIconId,
		// viewHolder.mIv_pic);
		// // 隐藏右上角图标
		// viewHolder.mIv_chose.setImageDrawable(new BitmapDrawable());
		// } else {
		String path = mPaths.get(position);

		// 设置ShowImageView的tag
		viewHolder.show.setTag(path);
		ImageLoaderWrapper.loadFromFile(viewHolder.show, path);
		// ImageLoader.getInstance().displayImage("file:///" + mPath,
		// viewHolder.show);
		// }

		// 设置SelectImageView的tag
		viewHolder.select.setTag(path + "select");
		// 处理滑动错位
		handleScrollBackgroundChange(position, viewHolder.show,
				viewHolder.select);
		return convertView;
	}

	/**
	 * 对外暴露图片管理器
	 */
	public SparseBooleanArray getSparseBooleanArray() {
		return mSparseManager;
	}

	/**
	 * 主要用来在选中操作时候，处理滑动错位的图片显示问题
	 */
	private void handleScrollBackgroundChange(int position, ImageView show,
			ImageView select) {
		int selectBtnBackground;
		int colorFilter;
		// if (position != 0) {
		if (isItemChecked(position)) {
			selectBtnBackground = mSelectedIcon;
			colorFilter = mSelectedColor;
		} else {
			selectBtnBackground = mUnSelectedIcon;
			colorFilter = mUnSelectedColor;
		}
		show.setColorFilter(colorFilter);
		// ImageLoader.getInstance().displayImage(
		// "drawable://" + selectBtnBackground, select);
		ImageLoaderWrapper.loadFromDrawable(select, selectBtnBackground);
		// } else {

		// }
	}

	private boolean isItemChecked(int position) {
		if (mSparseManager != null) {
			return mSparseManager.get(position);
		}
		return false;
	}

	/**
	 * 更新Adpter数据的方法
	 */
	public void update(List<String> list) {
		mPaths = list;
		mSparseManager.clear();// 特别注意清除所有保存的状态
		this.notifyDataSetChanged();
	}

	private class ViewHolder {
		ImageView show;
		ImageView select;
	}
}
```
<h3>6.附上ImageLoaderWrapper的代码</h3>

```
public class ImageLoaderWrapper {

	public static void loadFromFile(ImageView view, String 
	path) {
		ImageLoader.getInstance().displayImage("file:///" + path, view);
	}

	public static void loadFromDrawable(ImageView view, int drawableId) {
		ImageLoader.getInstance()
				.displayImage("drawable://" + drawableId, view);
	}
}
``` 

非常简单的代码。 是第三方库的UniversalImageLoader的包装。
[UniversalImageLoader Github](https://github.com/nostra13/Android-Universal-Image-Loader)

<h3>7.结束语</h3>
至此，完整的GalleryAdapter已经完毕了~希望还是要勤打代码，巩固知识！ 代码是最好的老师！

下面文章将讲解具体：
GalleryAdapter通过GalleryGridViewController实现与GalleryGridView的逻辑