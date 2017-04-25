---
title: 【选择图片Jar】全屏DialogFragment实现图片详情预览（十）
date: 2016-01-30 19:29
---
<h1>1.前言</h1>
上一篇文章我们在最后抛出了一个问题。到底我们要如何实现图片的详情页的预览。<!-- more -->
如图：

![这里写图片描述](http://img.blog.csdn.net/20160130144945530)


相信大家看了标题已经知道了，我们是用DialogFragment实现我们这个功能的！

嘿嘿，可能这个时候会有点惊讶，原来DialogFragment还有这样的效果？

奥秘在哪里呢？等下我们就会来揭晓。

再说两句~~Dialog和Fragment一直是我们开发中又爱又恨的两个东西，Dialog的Api繁多复杂，难以使用！ Fragment生命周期很难精准把握，而奇淫巧技非常多！ 

关于Dialog的问题，Google特意推出了DialogFragment来解决Dialog本身这个原生控件的弱点和不足，目前Google官方推荐使用DialogFragment，而且在实际开发中，发现的确是DialogFragment好用！

那关于Fragment的这个问题，我目前也在爬坑收集各种BUG和技巧中，之后学有所成，再回来分享吧！！因为这个不是三两句话可以完整概括的。 下面附上一条连接：，有兴趣的可以学习完本课程之后看看
[关于 Android，用多个 activity，还是单 activity 配合 fragment？-稀土掘金](http://gold.xitu.io/entry/56a2efe6816dfa005a8b0fa5/view)

<h1>2.FullScreenFragment布局实现</h1>
还是那个问题，我们不能打包资源文件进入Jar所以，还是要手动写代码，虽然过程比较繁琐复杂，但是还是要克服一下，这一次的痛苦，是为日后的快速打下基础！

```
public class FullScreenFragmentLayout extends RelativeLayout {
	/**
	 * Bar条
	 */
	private RelativeLayout mBar;
	/**
	 * 返回按钮的图片的id,删除的图片id,Bar条底色
	 */
	private int mBackIconId, mDeleteIconId, mBarColorId;
	/**
	 * Bar条中左边的返回ImageView,右边的返回ImageView
	 */
	private ImageView mBackIv, mDeleteIv;
	/**
	 * Bar条中的文字控件
	 */
	private TextView mNumTv;
	/**
	 * ViewPager空间
	 */
	private ViewPager mViewPager;
	/**
	 * Bar条高度
	 */
	private int mBarHeight;
	/**
	 * 图标的宽度
	 */
	private int mIconWidth;
	/**
	 * 图标距离边界的大小
	 */
	private int mIconMarginSide;
	/**
	 * 文字控件距离边界的大小
	 */
	private int mTextMarginSide;
	/**
	 * 文字大小
	 */
	private int mTextSize;
	/**
	 * 文字颜色
	 */
	private int mTextColorId;

	private FullScreenFragmentLayout(Context context, int mBackIconId,
			int mDeleteIconId, int mBarColorId, int mBarHeight, int mIconWidth,
			int mIconMarginSide, int mTextMarginSide, int mTextSize,
			int mTextColorId) {
		super(context);
		this.mBackIconId = mBackIconId;
		this.mDeleteIconId = mDeleteIconId;
		this.mBarColorId = mBarColorId;
		this.mBarHeight = mBarHeight;
		this.mIconWidth = mIconWidth;
		this.mIconMarginSide = mIconMarginSide;
		this.mTextMarginSide = mTextMarginSide;
		this.mTextSize = mTextSize;
		this.mTextColorId = mTextColorId;
		initViewPager();
		initBar();
	}

	private void initBar() {
		mBar = new RelativeLayout(getContext());
		if (mBarColorId != -1) {
			mBar.setBackgroundColor(getContext().getResources().getColor(
					mBarColorId));
		}

		ViewGroup.LayoutParams p = new ViewGroup.LayoutParams(
				ViewGroup.LayoutParams.MATCH_PARENT, mBarHeight);
		mBar.setLayoutParams(p);
		// bar.setBackgroundColor(getContext().getResources()
		// .getColor(mBarColorId));

		mBackIv = new ImageView(getContext());
		if (mBackIconId != -1) {
			mBackIv.setBackgroundResource(mBackIconId);
		}

		RelativeLayout.LayoutParams backParams = new RelativeLayout.LayoutParams(
				mIconWidth, mIconWidth);
		backParams.addRule(RelativeLayout.ALIGN_PARENT_LEFT);
		backParams.addRule(RelativeLayout.CENTER_VERTICAL);

		backParams.leftMargin = mIconMarginSide;
		mBackIv.setLayoutParams(backParams);
		mBar.addView(mBackIv);
		mNumTv = new TextView(getContext());
		// 注意 这里直接 无效
		// mNumTv.setTextColor("0xffffff");
		if (mTextColorId == -1) {
			mNumTv.setTextColor(Color.parseColor("#ffffff"));
		} else {
			mNumTv.setTextColor(getContext().getResources().getColor(
					mTextColorId));
		}
		mNumTv.setTextSize(mTextSize);
		RelativeLayout.LayoutParams numParams = new RelativeLayout.LayoutParams(
				RelativeLayout.LayoutParams.WRAP_CONTENT,
				RelativeLayout.LayoutParams.WRAP_CONTENT);
		numParams.addRule(RelativeLayout.ALIGN_PARENT_LEFT);
		numParams.addRule(RelativeLayout.CENTER_VERTICAL);
		int tvMargin = mTextMarginSide;
		numParams.leftMargin = tvMargin;
		mNumTv.setLayoutParams(numParams);
		mBar.addView(mNumTv);
		mDeleteIv = new ImageView(getContext());
		if (mDeleteIconId != -1) {
			mDeleteIv.setBackgroundResource(mDeleteIconId);
		}
		RelativeLayout.LayoutParams deleteParams = new RelativeLayout.LayoutParams(
				mIconWidth, mIconWidth);
		deleteParams.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
		deleteParams.addRule(RelativeLayout.CENTER_VERTICAL);
		deleteParams.rightMargin = mIconMarginSide;
		mDeleteIv.setLayoutParams(deleteParams);
		mBar.addView(mDeleteIv);
		this.addView(mBar);
	}

	private void initViewPager() {
		mViewPager = new ViewPager(getContext());
		ViewGroup.LayoutParams p = new ViewGroup.LayoutParams(
				ViewGroup.LayoutParams.MATCH_PARENT,
				ViewGroup.LayoutParams.MATCH_PARENT);
		mViewPager.setLayoutParams(p);
		this.addView(mViewPager);
	}

	public ViewPager getViewPager() {
		return mViewPager;
	}

	public TextView getNumberTextView() {
		return mNumTv;
	}

	public RelativeLayout getBar() {
		return mBar;
	}

	public static class Builder {
		private Context mContext;
		/**
		 * 返回按钮的图片的id,删除的图片id,Bar条底色
		 */
		private int mBackIconId, mDeleteIconId, mBarColorId;
		/**
		 * Bar条高度
		 */
		private int mBarHeight;
		/**
		 * 图标的宽度
		 */
		private int mIconWidth;
		/**
		 * 图标距离边界的大小
		 */
		private int mIconMarginSide;
		/**
		 * 文字控件距离边界的大小
		 */
		private int mTextMarginSide;
		/**
		 * 文字大小
		 */
		private int mTextSize;
		/**
		 * 文字颜色
		 */
		private int mTextColorId;

		public Builder(Context context) {
			mContext = context;
			// 初始化默认的大小
			initDefaultConfig();
		}

		private void initDefaultConfig() {
			// 获取屏幕高度
			int screenHeight = ScreenUtil.getScreenHeight(mContext);
			// 获取屏幕宽度
			int screenWidth = ScreenUtil.getScreenWidth(mContext);
			// bar高度为1/13
			mBarHeight = screenHeight / 13;

			mIconWidth = screenWidth / 13;

			mIconMarginSide = mIconWidth / 7;

			mTextSize = (int) (mIconMarginSide * 1.5);

			mTextMarginSide = mTextSize;

			mBackIconId = mBarColorId = mDeleteIconId = mTextColorId = -1;
		}

		public FullScreenFragmentLayout build() {
			return new FullScreenFragmentLayout(mContext, mBackIconId,
					mDeleteIconId, mBarColorId, mBarHeight, mIconWidth,
					mIconMarginSide, mTextMarginSide, mTextSize, mTextColorId);
		}

		public Builder setBarHeight(int height) {
			mBarHeight = height;
			return this;
		}

		public Builder setBarColorId(int colorId) {
			mBarColorId = colorId;
			return this;
		}

		public Builder setTextSize(int size) {
			mTextSize = size;
			return this;
		}

		public Builder setTextMarginLeftSize(int marginSize) {
			mTextMarginSide = marginSize;
			return this;
		}

		public Builder setTextColorId(int id) {
			mTextColorId = id;
			return this;
		}

		public Builder setIconSize(int size) {
			mIconWidth = size;
			return this;
		}

		public Builder setIconMarginSize(int marginSize) {
			mIconMarginSide = marginSize;
			return this;
		}

		public Builder setBackIconDrawableId(int id) {
			mBackIconId = id;
			return this;
		}

		public Builder setDeleteIconDrawableId(int id) {
			mDeleteIconId = id;
			return this;
		}
	}
}
```
这里我们采用了Builder模式进行创建，原因是因为可设置参数太多，采用这种模式比较方便创建。 

注意一下Builder构造方式里面对部分参数进行了默认大小处理，这是为了能够正常显示出来。

```
private void initDefaultConfig() {
			// 获取屏幕高度
			int screenHeight = ScreenUtil.getScreenHeight(mContext);
			// 获取屏幕宽度
			int screenWidth = ScreenUtil.getScreenWidth(mContext);
			// bar高度为1/13
			mBarHeight = screenHeight / 13;

			mIconWidth = screenWidth / 13;

			mIconMarginSide = mIconWidth / 7;

			mTextSize = (int) (mIconMarginSide * 1.5);

			mTextMarginSide = mTextSize;

			mBackIconId = mBarColorId = mDeleteIconId = mTextColorId = -1;
		}

```
即这段代码。 对id设置为-1表示标记该位没有设置值。

布局代码写起来很慢，也很繁琐。 所以要有耐心！ 如果还不太熟悉的话，可以查阅相关资料，这里就不一个一个来讲了！

<h1>3.FullScreenFragment的实现</h1>
OK!这里也是一个大工程！上好厕所， 拿好板凳围观吧。

首先来揭晓DialogFragment的奥秘：

```
	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		// 设置全屏
		setStyle(DialogFragment.STYLE_NO_TITLE,
				android.R.style.Theme_Black_NoTitleBar);
	}
```
就是这一行代码~ 具体的还有其他样式就不在这里讲述了。大家可以看看官方文档。
[官方文档：DialogFragment -不用翻墙也可以看！](http://cs.szpt.edu.cn/android/reference/android/app/DialogFragment.html)
那么下面我们看看构造方法：

```
	public static FullScreenFragment newInstance(int barColorId,
			int backIconId, int deleteIconId) {
		FullScreenFragment f = new FullScreenFragment();
		Bundle b = new Bundle();
		b.putInt(KEY_BACK_ICON_ID, backIconId);
		b.putInt(KEY_DELETE_ICON_ID, deleteIconId);
		b.putInt(KEY_BAR_COLOR_ID, barColorId);
		f.setArguments(b);
		return f;
	}

	public static FullScreenFragment newInstance(int barColorId, int barHeight,
			int backIconId, int deleteIconId, int iconWidth,
			int iconMarginSide, int textColor, int textSize,
			int textMarginLeftSize) {
		FullScreenFragment f = new FullScreenFragment();
		Bundle b = new Bundle();
		b.putInt(KEY_BAR_COLOR_ID, barColorId);
		b.putInt(KEY_BAR_HEIGHT, barHeight);
		b.putInt(KEY_BACK_ICON_ID, backIconId);
		b.putInt(KEY_DELETE_ICON_ID, deleteIconId);
		b.putInt(KEY_ICON_SIZE, iconWidth);
		b.putInt(KEY_ICON_MARGIN, iconMarginSide);
		b.putInt(KEY_TEXT_COLOR, textColor);
		b.putInt(KEY_TEXT_SIZE, textSize);
		b.putInt(KEY_TEXT_MARGIN, textMarginLeftSize);
		f.setArguments(b);
		return f;
	}

	private FullScreenFragment() {

	}
```
这里提供了两个构造方法。有的同学可能有疑问了，为什么要这样写呢？

其实我们可以从Google官方的很多代码看到这样的写法，通过setArguments(..);来传递信息,这是为了防止旋转之后，Android默认调用Fragment无参构造而产生的信息丢失。 具体的可以了解其他相关资料。

回到正题，其实上面那一堆东西都是为了从外部获取资源文件，或者样式，来创建我们FullScreenFragmentLayout的布局，这可以更加适应不同项目的需要。

下面我们来看看onCreateView方法：

```
	@Override
	public View onCreateView(LayoutInflater inflater, ViewGroup container,
			Bundle savedInstanceState) {
		// 创建布局
		mRootView = initLayout();
		// 获取控件
		mViewPager = mRootView.getViewPager();
		mNumberTv = mRootView.getNumberTextView();
		mBack = mRootView.getBackImageView();
		mDelete = mRootView.getDeleteImageView();

		// 初始化ViewPager
		initViewPager();
		// 改变文字标题
		postionChange(mCurrentPostion);
		// 要在设置完Adapter之后调用 不然没效果
		mViewPager.setCurrentItem(mCurrentPostion);
		// 初始化监听
		initBarListener();
		return mRootView;
	}
```
相信都很好理解。

下面贴上其他代码：

```
	private void initViewPager() {
		// 如果为空或者为0则返回
		if (mPaths == null || mPaths.size() == 0)
			return;
		mImageViews = new ImageView[mPaths.size()];
		for (int i = 0; i < mPaths.size(); i++) {
			mImageViews[i] = new ImageView(getActivity());
		}
		// mViewPager.setOffscreenPageLimit(0);
		mPagerAdapter = new PagerAdapter() {

			@Override
			public boolean isViewFromObject(View paramView, Object paramObject) {
				return paramView == paramObject;
			}

			@Override
			public int getCount() {
				return mPaths.size();
			}

			@Override
			public void destroyItem(ViewGroup container, int position,
					Object object) {
				((ViewPager) container).removeView(mImageViews[position]);
			}

			@Override
			public Object instantiateItem(ViewGroup container, int position) {
				ImageView i = mImageViews[position];
				ImageLoaderWrapper.loadFromFile(i, mPaths.get(position));
				((ViewPager) container).addView(i, 0);
				return i;
			}
		};

		mViewPager.setAdapter(mPagerAdapter);

		mViewPager.addOnPageChangeListener(new OnPageChangeListener() {

			@Override
			public void onPageSelected(int paramInt) {
				// 当前位置
				mCurrentPosition = paramInt;
				postionChange(mCurrentPosition);
				
			}

			@Override
			public void onPageScrolled(int paramInt1, float paramFloat,
					int paramInt2) {
				// 上一个位置 偏移量 下一个位置
			}

			@Override
			public void onPageScrollStateChanged(int paramInt) {
				// 这里是滑动的状态
			}
		});
	}

	/**
	 * 改变位置
	 * 
	 * @param position
	 */
	protected void postionChange(int position) {
		mNumberTv.setText(position + 1 + "/" + mPaths.size());
	}
	
		@Override
	public void onClick(View v) {
		String tag = (String) v.getTag();
		switch (tag) {
		case "back":
			this.dismiss();
			break;
		case "delete":
			// 删除
			break;
		}
	}
```
相信这些代码也难不倒大家。

稍微解释一下是，如果没有设置值就返回-1。这样可以用来方便构造我们的FullScreenFragmentLayout

还有的就是tag的用法，之前的文章也讲过，这里也不再说了~


然后我们要对外暴露两个方法，以便我们改变ViewPager显示的位置以及，所展示的图片列表：

```
	public void setList(List<String> list) {
		mPaths = list;
	}

	public void setCurrentPosition(int position) {
		mCurrentPostion = position;
	}
```

OK那再加上一些必要的东西，基本的FullScreenFragment就出来了。

```
public class FullScreenFragment extends DialogFragment implements
		OnClickListener {
	private static final String KEY_BAR_COLOR_ID = "bar_color";
	private static final String KEY_BAR_HEIGHT = "bar_height";
	private static final String KEY_BACK_ICON_ID = "back_icon";
	private static final String KEY_DELETE_ICON_ID = "delete_icon";
	private static final String KEY_ICON_MARGIN = "icon_margin";
	private static final String KEY_ICON_SIZE = "icon_size";
	private static final String KEY_TEXT_COLOR = "text_color";
	private static final String KEY_TEXT_SIZE = "text_size";
	private static final String KEY_TEXT_MARGIN = "text_margin";
	private FullScreenFragmentLayout mRootView;

	private ImageView mBack, mDelete;
	private ViewPager mViewPager;
	/**
	 * 数字的TextView
	 */
	private TextView mNumberTv;
	/**
	 * ImageView集合
	 */
	private ImageView[] mImageViews;
	/**
	 * 路径集合
	 */
	private List<String> mPaths;
	/**
	 * 当前位置
	 */
	private int mCurrentPostion = -1;

	private PagerAdapter mPagerAdapter;

	public static FullScreenFragment newInstance(int barColorId,
			int backIconId, int deleteIconId) {
		FullScreenFragment f = new FullScreenFragment();
		Bundle b = new Bundle();
		b.putInt(KEY_BACK_ICON_ID, backIconId);
		b.putInt(KEY_DELETE_ICON_ID, deleteIconId);
		b.putInt(KEY_BAR_COLOR_ID, barColorId);
		f.setArguments(b);
		return f;
	}

	public static FullScreenFragment newInstance(int barColorId, int barHeight,
			int backIconId, int deleteIconId, int iconWidth,
			int iconMarginSide, int textColor, int textSize,
			int textMarginLeftSize) {
		FullScreenFragment f = new FullScreenFragment();
		Bundle b = new Bundle();
		b.putInt(KEY_BAR_COLOR_ID, barColorId);
		b.putInt(KEY_BAR_HEIGHT, barHeight);
		b.putInt(KEY_BACK_ICON_ID, backIconId);
		b.putInt(KEY_DELETE_ICON_ID, deleteIconId);
		b.putInt(KEY_ICON_SIZE, iconWidth);
		b.putInt(KEY_ICON_MARGIN, iconMarginSide);
		b.putInt(KEY_TEXT_COLOR, textColor);
		b.putInt(KEY_TEXT_SIZE, textSize);
		b.putInt(KEY_TEXT_MARGIN, textMarginLeftSize);
		f.setArguments(b);
		return f;
	}

	private FullScreenFragment() {

	}

	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		// 设置全屏
		setStyle(DialogFragment.STYLE_NO_TITLE,
				android.R.style.Theme_Black_NoTitleBar);
	}

	@Override
	public View onCreateView(LayoutInflater inflater, ViewGroup container,
			Bundle savedInstanceState) {
		// if (mRootView == null) {
		// 创建布局
		mRootView = initLayout();
		// }
		// mViewPager = (ViewPager) view.findViewById(R.id.vp);
		// 获取空间
		mViewPager = mRootView.getViewPager();
		mNumberTv = mRootView.getNumberTextView();
		mBack = mRootView.getBackImageView();
		mDelete = mRootView.getDeleteImageView();
		// mPaths = new ArrayList<String>();

		// 初始化ViewPager
		initViewPager();
		// 改变文字标题
		postionChange(mCurrentPostion);
		// 要在设置完Adapter之后调用 不然没效果
		mViewPager.setCurrentItem(mCurrentPostion);
		// 初始化监听
		initBarListener();
		return mRootView;
	}

	private void initBarListener() {
		mBack.setTag("back");
		mDelete.setTag("delete");
		mBack.setOnClickListener(this);
		mDelete.setOnClickListener(this);
	}

	private FullScreenFragmentLayout initLayout() {
		Bundle b = getArguments();
		int barHeight = b.getInt(KEY_BAR_HEIGHT, -1);
		int barColorId = b.getInt(KEY_BAR_COLOR_ID, -1);
		int backIconId = b.getInt(KEY_BACK_ICON_ID, -1);
		int deleteIconId = b.getInt(KEY_DELETE_ICON_ID, -1);
		int iconSize = b.getInt(KEY_ICON_SIZE, -1);
		int iconMargin = b.getInt(KEY_ICON_MARGIN, -1);
		int textColorId = b.getInt(KEY_TEXT_COLOR, -1);
		int textSize = b.getInt(KEY_TEXT_SIZE, -1);
		int textMargin = b.getInt(KEY_TEXT_MARGIN, -1);
		FullScreenFragmentLayout.Builder builder = new FullScreenFragmentLayout.Builder(
				getActivity());
		if (barHeight != -1) {
			builder.setBarHeight(barHeight);
		}
		if (barColorId != -1) {
			builder.setBarColorId(barColorId);
		}
		if (backIconId != -1) {
			builder.setBackIconDrawableId(backIconId);
		}
		if (deleteIconId != -1) {
			builder.setDeleteIconDrawableId(deleteIconId);
		}
		if (iconSize != -1) {
			builder.setIconSize(iconSize);
		}
		if (iconMargin != -1) {
			builder.setIconMarginSize(iconMargin);
		}
		if (textColorId != -1) {
			builder.setTextColorId(textColorId);
		}
		if (textSize != -1) {
			builder.setTextSize(textSize);
		}
		if (textMargin != -1) {
			builder.setTextMarginLeftSize(textMargin);
		}
		return builder.build();
	}

	@Override
	public void onViewCreated(View view, Bundle savedInstanceState) {
		super.onViewCreated(view, savedInstanceState);
	}

	private void initViewPager() {
		// 如果为空或者为0则返回
		if (mPaths == null || mPaths.size() == 0)
			return;
		System.out.println("initViewPager Context = " + getActivity());
		mImageViews = new ImageView[mPaths.size()];
		for (int i = 0; i < mPaths.size(); i++) {
			mImageViews[i] = new ImageView(getActivity());
		}
		mViewPager.setOffscreenPageLimit(2);
		mPagerAdapter = new PagerAdapter() {
			@Override
			public boolean isViewFromObject(View paramView, Object paramObject) {
				// 判断当前是否一致，一致则绘制View出来
				return paramView == paramObject;
			}

			@Override
			public int getCount() {
				return mPaths.size();
			}

			@Override
			public void destroyItem(ViewGroup container, int position,
					Object object) {
				System.out.println("destroyItem: viewPager = "
						+ mViewPager.toString() + " container = " + container
						+ " obj = " + object);
				// ((ViewPager) container).removeView(mImageViews[position]);
				View v = (View) object;
				// String vTag = (String) v.getTag();
				// if (mMovedPath != null && vTag.equals(mMovedPath)) {
				((ViewPager) container).removeView(v);
				v = null;
				// }
			}

			@Override
			public Object instantiateItem(ViewGroup container, int position) {
				System.out.println("destroyItem: viewPager = "
						+ mViewPager.toString() + " container = " + container);
				ImageView i = mImageViews[position];
				String path = mPaths.get(position);
				i.setTag(path);
				ImageLoaderWrapper.loadFromFile(i, path);
				((ViewPager) container).addView(i);
				System.out.println("instantiateItem");
				return i;
			}

			@Override
			public int getItemPosition(Object object) {
				System.out.println("getItemPosition: object =" + object);
				return POSITION_NONE;
			}

			@Override
			public void notifyDataSetChanged() {
				System.out.println("notifyDataSetChanged");
				super.notifyDataSetChanged();
			}

			@Override
			public void startUpdate(ViewGroup container) {

			}

			@Override
			public void finishUpdate(ViewGroup container) {

			}
		};

		mViewPager.setAdapter(mPagerAdapter);

		mViewPager.addOnPageChangeListener(new OnPageChangeListener() {

			@Override
			public void onPageSelected(int paramInt) {
				// 当前位置
				mCurrentPostion = paramInt;
				postionChange(mCurrentPostion);
			}

			@Override
			public void onPageScrolled(int paramInt1, float paramFloat,
					int paramInt2) {
				// 上一个位置 偏移量 下一个位置
			}

			@Override
			public void onPageScrollStateChanged(int paramInt) {
				// 这里是滑动的状态
			}
		});
	}

	/**
	 * 改变位置
	 * 
	 * @param position
	 */
	protected void postionChange(int position) {
		mNumberTv.setText(position + 1 + "/" + mPaths.size());
	}

	public void setList(List<String> list) {
		mPaths = list;
	}

	public void setCurrentPosition(int position) {
		mCurrentPostion = position;
	}

	@Override
	public void onClick(View v) {
		String tag = (String) v.getTag();
		switch (tag) {
		case "back":
			this.dismiss();
			break;
		case "delete":

			break;
		}
	}
}
```
OK我们继续完善一下逻辑，我们还有删除这个逻辑没有实现，具体的逻辑如下：

```
	@Override
	public void onClick(View v) {
		....
		case "delete":
			mPaths.remove(mCurrentPostion);
			mPagerAdapter.notifyDataSetChanged();
			mCurrentPostion = mViewPager.getCurrentItem();
			postionChange(mCurrentPostion);
			break;
		}
		...
	}
```
至此是不是以为已经完成了？ 没有哦！ 因为我们还要通知外面的那个展示列表更新哦！
所以我们需要在这个DialogFragment dismiss的时候判断一下是否需要对外部进行更新！

我们依旧采用接口回调的思想：

```
	@Override
	public void onClick(View v) {
		....
		case "delete":
			if (!mNeedToUpdate) {
				mNeedToUpdate = true;
			}
			mPaths.remove(mCurrentPostion);
			if (mPaths.size() == 0) {
				dismiss();
				return;
			}
			mPagerAdapter.notifyDataSetChanged();
			mCurrentPostion = mViewPager.getCurrentItem();
			postionChange(mCurrentPostion);
			break;
		}
		...
	}
	
	@Override
	public void dismiss() {
		super.dismiss();
		if (mNeedToUpdate) {
			mCallback.update(mPaths);
		}
	}

	public void setUpdateCallback(UpdateCallback callback) {
		mCallback = callback;
	}

	public interface UpdateCallback {
		void update(List<String> mPaths);
	}
```
在DialogFragment消失的时候，通知他去更新！

最后我们还要监听一下用户的back实体键：

```
	@Override
	public void onViewCreated(View view, Bundle savedInstanceState) {
		super.onViewCreated(view, savedInstanceState);
		this.getDialog().setOnKeyListener(new OnKeyListener() {
			public boolean onKey(DialogInterface dialog, int keyCode,
					KeyEvent event) {
				if (keyCode == KeyEvent.KEYCODE_BACK) {
					dismiss();
					return true;
				} else {
					return false;
				}
			}
		});
	}
```
当用户按下back键的时候，我们也让它dissmiss()

OK！至此FullScreenFragment就这样完成啦！！

那么我们现在看看Controller的代码！

<h1>4.完整的ChosenPhotoViewController</h1>
那么Controller中的click代码就可以变成：

```
	....
	@Override
	public void click(int position) {
		if (position == mPaths.size()) {
			// 跳转逻辑
			....
		} else {
			if (mFullFragment == null) {
				throw new IllegalArgumentException(
						"FullScreenFragment is null. you should invoke setDetailsFragment first");
			}
			if (mManager == null) {
				throw new IllegalArgumentException(
						"FragmentManager is null. you should invoke setDetailsFragment first");
			}
			mFullFragment.setList(mPaths);
			mFullFragment.setCurrentPosition(position);
			mFullFragment.show(mManager, "FullFragment");
		}
	}
	....
```
这里需要**注意一下** DialogFragment的show是**异步**的！**异步**的！**异步**的！

当我们调用show的时候，不是立即显示出来的。 
[Android DialogFragment getActivity() is null -StackOverFlow](http://stackoverflow.com/questions/22490687/android-dialogfragment-getactivity-is-null)
 
 所以，在DialogFragment中，我们对外暴露的方法里面，一定不能涉及上下文的操作，也不能涉及View相关的代码！

这个坑我在开发的时候花了好长的时候，一开始我是在

```
mFullFragment.setList(mPaths);
```
才开始创建PagerAdapter，然后当我调用

```
mViewPager.setAdapter(mAdapter)
```
死活报的都是空指针异常

所以要好好注意这里！

到这里还没有完哦，我们还要在Controller中加一个对外暴露的方法，传如FullScreenFragment实例，以及FragmentManager

```
public void setDetailsFragment(@NonNull FragmentManager manager,
	@NonNull FullScreenFragment fragment) {
		mFullFragment = fragment;
		mManager = manager;
		//设置回调监听！
		mFullFragment.setUpdateCallback(new UpdateCallback() {

			@Override
			public void update(List<String> paths) {
				mAdapter.update(paths);
			}
		});
}

```

OK!那现在完整的ChosenPhotoController的完整代码也呼之欲出啦！

```
public class ChosenPhotoViewController extends BaseController {
	/**
	 * 一行图片的数量
	 */
	private int mNumColumuns;
	/**
	 * +号图片的资源Id
	 */
	private int mAddIconId;
	/**
	 * 图片路径
	 */
	private List<String> mPaths;
	/**
	 * +号图片，接口回调
	 */
	private AddIconAction mAction;
	private IPhotoGridView mView;
	private ChosenPhotoAdapter mAdapter;

	private FragmentManager mManager;
	private FullScreenFragment mFullFragment;

	public ChosenPhotoViewController(int numColumns, int addIconId) {
		mAddIconId = addIconId;
		mNumColumuns = numColumns;
		mPaths = new ArrayList<String>();
	}

	/**
	 * 设置监听，主要是Controller和View联动使用 不需要手动调用
	 * 
	 * @param listener
	 */
	@Override
	public void setPhotoViewListener(IPhotoGridView view) {
		mView = view;
		mAdapter = new ChosenPhotoAdapter(mView.getGridView().getContext(),
				mPaths, mNumColumuns, mAddIconId);
		mView.bindAdapter(mAdapter);
	}

	@Override
	public List<String> getPaths() {
		return mPaths;
	}

	@Override
	public void setAdapter(List<String> paths) {
		if (mView == null)
			throw new IllegalArgumentException(
					"mListener is null. you should invoke ChosenPhotoGridView's bindController() first");
		mPaths = paths;
		mAdapter.update(mPaths);
	}

	@Override
	public void click(int position) {
		if (position == mPaths.size()) {
			if (mAction == null) {
				throw new IllegalArgumentException(
						"AddIconAction is null. you should invoke setAddIconAction first");
			}
			mAction.onAddClick();
		} else {
			if (mFullFragment == null) {
				throw new IllegalArgumentException(
						"FullScreenFragment is null. you should invoke setDetailsFragment first");
			}
			if (mManager == null) {
				throw new IllegalArgumentException(
						"FragmentManager is null. you should invoke setDetailsFragment first");
			}
			mFullFragment.setList(mPaths);
			mFullFragment.setCurrentPosition(position);
			mFullFragment.show(mManager, "FullFragment");
		}
	}

	public void setDetailsFragment(@NonNull FragmentManager manager,
			@NonNull FullScreenFragment fragment) {
		mFullFragment = fragment;
		mManager = manager;
		mFullFragment.setUpdateCallback(new UpdateCallback() {

			@Override
			public void update(List<String> paths) {
				mAdapter.update(paths);
			}
		});
	}

	public void setAddIconAction(AddIconAction action) {
		mAction = action;
	}

	public interface AddIconAction {
		void onAddClick();
	}
}
```
到时候我们使用起来就是这样：

```
mGv = (GalleryGridView) findViewById(R.id.gv);

mController = new ChosenPhotoViewController(4, R.drawable.add);

FullScreenFragment f =FullScreenFragment.newInstance
(R.color.bar,R.drawable.btn_return, R.drawable.btn_delete);
		mController.setDetailsFragment(getSupportFragmentManager(), f);
		
mController.setAddIconAction(new AddIconAction() {
	@Override
	public void onAddClick() {
		Intent i = new Intent(ShowActivity.this, MainActivity.class);
		startActivityForResult(i, 1);
	}
});

mGv.bindController(mController);
```
嗯，代码还算优雅，也还算比较少！

<h1>5.结束语</h1>

OK! 现在这个Jar要已经达到了我们日常项目开发所需要的样子了！！！



