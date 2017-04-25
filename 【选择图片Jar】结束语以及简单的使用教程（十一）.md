---
title: 【选择图片Jar】结束语以及简单的使用教程（十一）
date: 2016-01-31 17:32
---
<h1>1.前言</h1>
OK 经过前面这么多篇文章的讲解，相信大家已经学到了不少的东西！那么接下来分享几个封装好的工具！
<!-- more -->
<h1>2.PhotoScanner图片扫描</h1>

```
public class PhotoScanner {
	private static PhotoScanner mScanner;
	private Context mContext;
	/**
	 * 图片的父亲路径集合
	 */
	private List<String> mParentPaths = new ArrayList<String>();
	/**
	 * 父亲路径对应的文件夹内的路径的Map集合 key为parentPath,value为不带parentPath的不完整图片路径
	 */
	private HashMap<String, List<String>> mImgsPaths;
	/**
	 * 第一张图片的完整路径
	 */
	private List<String> mFirstPicPaths;
	/**
	 * 所有文件夹中图片的数量 key为parentPath value为文件夹中图片的数量
	 */
	private HashMap<String, Integer> mPathsNumMap;
	/**
	 * 文件过滤器
	 */
	private FilenameFilter mFloderFilter;

	public static PhotoScanner getInstance(Context context) {
		if (mScanner == null) {
			synchronized (PhotoScanner.class) {
				if (mScanner == null) {
					mScanner = new PhotoScanner(context);
				}
			}
		}
		return mScanner;
	}

	public PhotoScanner(Context context) {
		mContext = context;
		mParentPaths = new ArrayList<String>();
		mImgsPaths = new HashMap<String, List<String>>();
		mFirstPicPaths = new ArrayList<String>();
		mPathsNumMap = new HashMap<String, Integer>();
		mFloderFilter = new FilenameFilter() {
			@Override
			public boolean accept(File dir, String filename) {
				if (filename.endsWith(".jpg") || filename.endsWith(".jpeg")
						|| filename.endsWith(".png")) {
					return true;
				}
				return false;
			}
		};
	}

	public synchronized void scan() {
		ContentResolver cr = mContext.getContentResolver();
		Uri uri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI;
		Cursor cursor = cr.query(uri, null, MediaStore.Images.Media.MIME_TYPE
				+ "=? or " + MediaStore.Images.Media.MIME_TYPE + "=?",
				new String[] { "image/jpeg", "image/png" },
				MediaStore.Images.Media.DATE_MODIFIED);
		if (cursor.moveToFirst()) {
			do {
				String path = cursor.getString(cursor
						.getColumnIndex(MediaStore.Images.Media.DATA));
				File childFile = new File(path);// 当前图片文件
				File parentFile = childFile.getParentFile();// 获取文件夹
				String parentPath = parentFile.getAbsolutePath();// 获取文件夹路径
				if (mParentPaths.contains(parentPath)) {
					continue;// 如果已经有了这个父亲路径文件夹，说明该文件夹内的图片已经全部获取了。
				} else {
					mParentPaths.add(parentPath); // 添加到父亲路径中
				}
				List<String> childImgsWithParentPath = Arrays.asList(parentFile
						.list(mFloderFilter));// 该文件夹中的所有图片集合
				mImgsPaths.put(parentPath, childImgsWithParentPath);// 添加到Map中
				mFirstPicPaths.add(path);// 首张图片路径获取
				mPathsNumMap.put(parentPath, childImgsWithParentPath.size());// 文件夹中图片数量获取
			} while (cursor.moveToNext());
		}
		cursor.close();
	}

	/**
	 * 获取所有父亲路径
	 * 
	 * @return
	 */
	public List<String> getParentPaths() {
		return mParentPaths;
	}

	/**
	 * 获取所有图片路径 key为父亲路径，value为该路径下的图片集合
	 * 
	 * @return
	 */
	public HashMap<String, List<String>> getImgsPaths() {
		return mImgsPaths;
	}

	/**
	 * 获取第一张图片路径
	 * 
	 * @return
	 */
	public List<String> getFirstPicPaths() {
		return mFirstPicPaths;
	}

	/**
	 * 获取文件夹内的图片数量
	 * 
	 * @return
	 */
	public HashMap<String, Integer> getPathsNumMap() {
		return mPathsNumMap;
	}

}
```

<h1>3.VideoScanner视频扫描</h1>

```
public class VideoScanner {
	private static VideoScanner mScanner;
	private Context mContext;
	private List<VideoMessage> mVideoList;

	public static VideoScanner getInstance(Context context) {
		if (mScanner == null) {
			synchronized (VideoScanner.class) {
				if (mScanner == null) {
					mScanner = new VideoScanner(context);
				}
			}
		}
		return mScanner;
	}

	private VideoScanner(Context context) {
		mContext = context;
		mVideoList = new ArrayList<VideoMessage>();
	}

	public synchronized void scan() {
		ContentResolver cr = mContext.getContentResolver();
		Uri uri = MediaStore.Video.Media.EXTERNAL_CONTENT_URI;
		Cursor cursor = cr.query(uri, null, MediaStore.Video.Media.MIME_TYPE
				+ "=? or " + MediaStore.Video.Media.MIME_TYPE + "=? or "
				+ MediaStore.Video.Media.MIME_TYPE + "=?", new String[] {
				"video/mp4", "video/avi", "video/x-matroska" },
				MediaStore.Video.Media.DEFAULT_SORT_ORDER);
		if (cursor.moveToFirst()) {
			do {
				String path = cursor.getString(cursor
						.getColumnIndex(MediaStore.Video.Media.DATA));
				String title = cursor.getString(cursor
						.getColumnIndex(MediaStore.Video.Media.TITLE));
				int id = cursor.getInt(cursor
						.getColumnIndex(MediaStore.Video.Media._ID));
				mVideoList.add(new VideoMessage(id, title, path));
			} while (cursor.moveToNext());
		}
		cursor.close();
	}

	public List<VideoMessage> getVideoList() {
		return mVideoList;
	}
}
```
代码都比较简单，就不多说了。我们可以将他一并放入Jar中。

<h1>4.使用</h1>
这两个工具类的使用，以PhotoScanner为例：

```
PhotoScanner scanner = PhotoScanner
						.getInstance(MainActivity.this);
```
然后调用

```
scanner.scan();
```
然后我们就可以通过get方法返回我们需要的列表了！

不过需要注意，scan()内部实现大家也看到了是查询系统数据库，所以建议开线程获取！

附上选择图片Activity:

```
public class GalleryActivity extends Activity {
	private GalleryGridView mGridView;
	private List<String> mPhotoes;
	private Handler mHandler = new Handler();
	private GalleryViewController mController;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);

		setContentView(R.layout.activity_main);

		mGridView = (GalleryGridView) findViewById(R.id.gv);

		mController = new GalleryViewController(
				GalleryViewController.MUTIPLE_MODE, 9,
				R.drawable.pictures_selected, R.drawable.picture_unselected);
		mGridView.bindController(mController);
		mPhotoes = new ArrayList<String>();
		getPhotoes();

		findViewById(R.id.btn).setOnClickListener(new OnClickListener() {

			@Override
			public void onClick(View v) {
				ArrayList<String> list = (ArrayList<String>) mController
						.getPaths();
				Intent i = getIntent();
				Bundle b = new Bundle();
				b.putStringArrayList("list", list);
				i.putExtra("photo", b);
				setResult(1, i);
				GalleryActivity.this.finish();
			}
		});
	}

	private void getPhotoes() {
		new Thread(new Runnable() {
			@Override
			public void run() {
				PhotoScanner scanner = PhotoScanner
						.getInstance(GalleryActivity.this);
				scanner.scan();
				HashMap<String, List<String>> photosMap = scanner
						.getImgsPaths();
				for (String parentPath : photosMap.keySet()) {
					// StringBuilder sb = new StringBuilder(parentPath);
					for (String childPath : photosMap.get(parentPath)) {
						// 这里需要特别注意"/"的拼接
						mPhotoes.add(parentPath + "/" + childPath);
					}
				}
				// VideoScanner scanner = VideoScanner
				// .getInstance(MainActivity.this);
				// scanner.scan();
				// List<VideoMessage> list = scanner.getVideoList();
				// for (VideoMessage v : list) {
				// mPhotoes.add(v.getPath());
				// }
				mHandler.post(new Runnable() {
					@Override
					public void run() {
						mController.setAdapter(mPhotoes);
					}
				});
			}
		}).start();
	}
}
```
局部文件：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical" >

    <com.hans.selectphoto.view.GalleryGridView
        android:id="@+id/gv"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
       
        
        android:numColumns="3" />

    <Button
        android:id="@+id/btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```

再附上 展示图片Activity：

```
public class ShowActivity extends FragmentActivity {
	private GalleryGridView mGv;
	private Button mBtn;
	private ChosenPhotoViewController mController;
	private List<String> mPaths;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		// TODO Auto-generated method stub
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_show);
		mGv = (GalleryGridView) findViewById(R.id.gv);

		mController = new ChosenPhotoViewController(4, R.drawable.add);
		
		FullScreenFragment f = FullScreenFragment.newInstance(R.color.bar,
				R.drawable.btn_return, R.drawable.btn_delete);
			
		mController.setDetailsFragment
				(getSupportFragmentManager(), f);

		mController.setAddIconAction(new AddIconAction() {

			@Override
			public void onAddClick() {
				Intent i = new Intent(ShowActivity.this, GalleryActivity.class);
				startActivityForResult(i, 1);
			}
		});
		
		mGv.bindController(mController);
	}

	@Override
	protected void onActivityResult(int requestCode, int resultCode, Intent data) {
		super.onActivityResult(requestCode, resultCode, data);
		if (requestCode == 1 && resultCode == 1) {
			Bundle b = data.getBundleExtra("photo");
			List<String> list = b.getStringArrayList("list");
			mPaths = new ArrayList<String>();
			mPaths.addAll(list);
			Toast.makeText(this, list.toString(), 1000).show();
			mController.setAdapter(list);
		}
	}
}
``` 
布局文件：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical" >

    <com.hans.selectphoto.view.GalleryGridView
        android:id="@+id/gv"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_marginLeft="5dp"
        android:layout_marginRight="5dp"
        android:layout_weight="1"
        android:horizontalSpacing="5dp"
        android:numColumns="4"
        android:verticalSpacing="5dp" />

</LinearLayout>
```

相信代码还是比较简单的！ 使用起来还是比较方便！

需要注意：
项目还要额外依赖：[UniversalImageLoader-Github
](https://github.com/nostra13/Android-Universal-Image-Loader)

添加ImageLoader所需权限：

```
 <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
 <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

以及配置MApplication，这些UniversalImageLoader文档都有说。

<h1>5.源代码以及Jar下载</h1>
[源代码以及Jar](https://github.com/AlphaHans/SelectPhoto)

下载整个Zip之后，Jar可以从../SelectPhotoJar中取出

OK！本系列课程就到此结束了！！