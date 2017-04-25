---
title: 【Weex】Android集成Weex
date: 2017-01-18 22:00
tag: Weex
category: Weex
---

## 添加依赖

导入依赖

<!-- more -->

```

compile 'com.android.support:support-v4:23.1.1'
compile 'com.android.support:design:23.1.1'
compile 'com.android.support:support-annotations:23.1.1'
compile 'com.android.support:appcompat-v7:23.1.1'
compile 'com.android.support:recyclerview-v7:23.1.1'
compile 'com.alibaba:fastjson:1.1.46.android'
compile 'com.squareup.picasso:picasso:2.5.2'
compile 'com.taobao.android:weex_sdk:0.9.4' //添加SDK library依赖
```

## 放入Weex的JS文件

然后拿到我们Hello.js的文件，在目录的weex_temp下

![](http://p1.bqimg.com/567571/91b5d4c86813768b.png)

放入到工程的Assets中

![](http://p1.bqimg.com/567571/c3b68cbe17c9b5f7.png)



## 配置Weex到Android

* 初始化Weex组件

```

public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        WXSDKEngine.init(this,null,null,new MyImageAdapter(),null);//设置自定义的adadpter实现图片显示、http请求等能力
    }
}
```

* 渲染组件

```

public class MainActivity extends AppCompatActivity implements IWXRenderListener {
    public static final String TAG = "MainActivity";
    WXSDKInstance mInstance;
    ViewGroup mContainer;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mContainer = (ViewGroup) findViewById(R.id.container);
        mInstance = new WXSDKInstance(this); //create weex instance
        mInstance.registerRenderListener(this); //SimpleRenderListener需要开发者来实现

        //sample直接使用Weex命令转化后的js代码,也可以从文件加载或者从服务端下载的方式
        mInstance.render(TAG,
                WXFileUtils.loadAsset("hello.js",this),
                new HashMap<String, Object>(),
                null,
                ScreenUtil.getDisplayWidth(this),
                ScreenUtil.getDisplayHeight(this),
                WXRenderStrategy.APPEND_ASYNC);
    }

    @Override
    public void onStart() {
        super.onStart();
        if (mInstance != null) {
            mInstance.onActivityStart();
        }
    }

    @Override
    public void onResume() {
        super.onResume();
        if (mInstance != null) {
            mInstance.onActivityResume();
        }
    }

    @Override
    public void onPause() {
        super.onPause();
        if (mInstance != null) {
            mInstance.onActivityPause();
        }
    }

    @Override
    public void onStop() {
        super.onStop();
        if (mInstance != null) {
            mInstance.onActivityStop();
        }
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mInstance != null) {
            mInstance.onActivityDestroy();
        }
    }

    @Override
    public void onViewCreated(WXSDKInstance wxsdkInstance, View view) {
        if (mContainer != null) {
            mContainer.addView(view);
        }
    }

    @Override
    public void onRenderSuccess(WXSDKInstance wxsdkInstance, int i, int i1) {

    }

    @Override
    public void onRefreshSuccess(WXSDKInstance wxsdkInstance, int i, int i1) {

    }

    @Override
    public void onException(WXSDKInstance wxsdkInstance, String s, String s1) {

    }
}
```

需要特别注意这一段代码：

```

    @Override
    public void onViewCreated(WXSDKInstance wxsdkInstance, View view) {
        if (mContainer != null) {
            mContainer.addView(view);
        }
    }
```



## 运行结果

![](http://p1.bpimg.com/567571/6e980c59a017b798.png)



至此则完成了Weex的Demo



