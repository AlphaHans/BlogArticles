---
title: 【MVP】我眼中的MVP
date: 2016-04-18 20:00
tags: [项目重构,MVP]  
category: Android进阶
---

<!-- more -->
# 致敬Kobe
开头先致敬Kobe，感谢他所带来的一切，永远是我的MVP（Most Valuable Player）
![Kobe](http://img.blog.csdn.net/20160418174652082)

# 我的MVP学习历程
近两年，Android已经越来越成熟，无论是UI设计，还是各种各样的库；随着项目的越来越大，项目架构方面探索也越来越成熟了。

从去年6月就接触了MVP，发现这的确是一个很不错的项目架构方式。 从一脸茫然，到有自己的一些感悟，希望自己也能够记录下来。

本文并不讲MVP具体到底是干嘛的，网上的扫盲贴已经很多了。 本文主要讲自己使用MVP的心路历程。

# 初出茅庐使用MVP

## 1.项目使用
早期的一个项目：

Actvitiy：
![这里写图片描述](http://img.blog.csdn.net/20160418200811068)
Fragment：
![这里写图片描述](http://img.blog.csdn.net/20160418233141355)
MVP#Model：（主要指的是实体类）
![这里写图片描述](http://img.blog.csdn.net/20160418200837711)
MVP#View：
![这里写图片描述](http://img.blog.csdn.net/20160418200848414)
MVP#Presenter：
![这里写图片描述](http://img.blog.csdn.net/20160418200856007)

列出来的目的就是说，实际上采用MVP模式是会增加不少代码量的。

以登陆的例子来说：
```
public class LoginActivity extends BaseActivity implements View.OnClickListener, ILoginActivityView {
    private LoginActivityPresenter mPresenter;
    private AppCompatEditText mUserName;
    private AppCompatEditText mPsw;
    private TopBar mTopBar;
    private DialogUtil mDialogUtil;

    @Override
    protected int getLayoutId() {
        return R.layout.activity_login;
    }

    @Override
    protected int[] getItemsId() {
        return new int[]{R.id.login_btn_login, R.id.login_tv_register};
    }

    @Override
    protected void initDate() {
        mPresenter = new LoginActivityPresenter(this, this);
    }

    @Override
    protected void initView() {
        mUserName = (AppCompatEditText) findViewById(R.id.login_tv_usr_name);
        mPsw = (AppCompatEditText) findViewById(R.id.login_tv_psw);

        mTopBar = (TopBar) findViewById(R.id.top_bar);
        mTopBar.setLeftImageDrawable(R.drawable.btn_arrow_left);
        mTopBar.setRightButtonVisibility(false);
        mTopBar.setTitle("登陆");
        mDialogUtil = new DialogUtil(this);
    }

    //...

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.login_btn_login:
                mPresenter.login(mUserName.getText().toString(),mPsw.getText().toString());
                break;
            case R.id.login_tv_register:
                mPresenter.toRegisterActivity();
                break;
        }
    }

    @Override
    public void showLoginLoading() {
        if (mDialogUtil == null) mDialogUtil = new DialogUtil(this);
        mDialogUtil.showLoadingDialog();
    }

    @Override
    public void hideLoginLoading() {
        if (mDialogUtil != null) {
            mDialogUtil.dismissLoadingDialog();
        }
    }

    //...
}
```
只是直接new了Presenter，而且传入了context和View接口ILoginActivityView
```
public interface ILoginActivityView {
    void showLoginLoading();
    void hideLoginLoading();
    void success();
    void error();
}
```
Presenter代码：
```
public class LoginActivityPresenter {
    private Context mContext;
    private ILoginActivityView mView;
    private HttpVolley mVolley;
    private LoginActivityEntity mEntity;

    public LoginActivityPresenter(Context context, ILoginActivityView view) {
        this.mContext = context;
        this.mView = view;
        mVolley = HttpVolley.getInstance(context);
    }

    public void login(String name,String psw) {
        if (mEntity != null) return;
        mView.showLoginLoading();
        LoginHelper.getInstance(mContext).login(name, psw, new LoginHelper.LoginListener() {
            @Override
            public void onSuccess(LoginActivityEntity entity) {
                mEntity = entity;
                mView.hideLoginLoading();
                mView.success();
            }

            @Override
            public void onError(String error) {
                mView.hideLoginLoading();
                mView.error();
            }
        });
    }
}
```
## 2.项目感悟
### 1.缺点
1.代码量会增加不少
2.Presenter引用了Context对象，实际上又增加了耦合度
### 2.优点
1.逻辑和界面可以很好地分开开来
2.面向接口编程，不依拉具体实现。所以只要将View内的操作敲定，Presenter就可以开始编码，不需要依赖Activity的布局等等之类的。 开发效率高一些

# 再战MVP
实验室又有其他的新项目，经过上一次的总结。所以的话，我还是想使用MVP来开发这个项目。

接受教训，所以这次先定义了Base基础类。

## 1.BaseView
其实IView在实际业务逻辑中，无非是有List列表和无List两种。 所以我定义了两个BaseView
```
public interface BaseView {
    public Context getMyContext();//无论如何逃避，在一些特殊的情况，Presenter还是需要引用Context对象

    public void showLoading();//提升交互，开始提示加载框

    public void hideLoading();//加载框消失

    public void showSuccess();//网络或本地数据请求成功之后的回调

    public void showError();//失败回调

    public void showEmpty();//空数据回调
}
```
而带有List界面的，则需要添加多这四个接口
```
public interface BaseListView extends BaseView {
    public void addList(List data);//列表数据请求成功，加入到Adapter数据，刷新界面

    public void refreshComplete();//刷新数据回调

    public void loadMoreComplete();//加载更多数据回调

    public void showNoMoreData();//没有更多数据回调
}
```
上面这两个基础的View,已经涵盖了大部分业务开发的场景，所以需要加具体的逻辑，直接继承他们其中一个，然后添加自己的接口方法即可。

## 2.BasePresenter
```
public abstract class BasePresenter<T extends BaseView> {
    protected T mView;
    protected Bundle mData;

    public BasePresenter(T view) {
        mView = view;
    }

    public void create(Bundle data) { //这里主要是将Activity中onCreate的Bundle传递过来
        mData = data;
    }

    abstract public void start();//数据加载

    abstract public void update();//数据刷新

    public void destory() {
        mView = null;//释放对View的引用，避免造成内存泄漏
    }
}
```
实际上在BasePresenter这里需要特别注意的问题就是对实现了View接口的Activity保持了引用，这样会造成内存泄露。 所以一定要在Activity销毁之后，也要调用Presenter释放对View的引用。

在这里实际上有衍生的写法，加入一个方法：
```
public void onBindView(T view) {
    mView  = view;
}
```
在Activity销毁的时候调用：
```
mPresenter.onBindView(null); 
```
这个写法，对于在允许旋转Activity的项目中更有利，这句话可能不明白是什么意思。

因为Activity旋转之后会重新创建，但Presenter是没有必要重新创建的！比如我Presenter持有原来从网络请求回来的数据，那么就可以直接给这个新的Activity使用了，不需要再进行网络请求。 

假设在保持了Presenter单例的情况下，Presenter会持有原来的Activity的接口View引用，造成内存泄漏（也就是上面那个情况），那么这个时候 onBindView这个方法的用处就来了。 直接调用onBindView，就可以和新的Activity的View接口绑定上了，释放了原来的引用，也解决了内存泄漏的问题。

## 3.BaseActivity
```
public abstract class BaseActivity<T extends BasePresenter> extends AppCompatActivity {
    protected T mPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getContentViewId());
        ButterKnife.bind(this);
        mPresenter = getPresenter();
        init();
    }


    abstract protected int getContentViewId();

    abstract protected T getPresenter();

    abstract protected void init();


    @Override
    protected void onDestroy() {
        super.onDestroy();
        ButterKnife.unbind(this);
        if (mPresenter != null)
            mPresenter.destory();
    }
}
```
这里BaseActivity的设计，直接将Presenter和Activity绑定起来，在Activity销毁的时候就可以直接自动解除。

**说明一下：
1.实验室项目一般都是强制竖屏，所以没考虑旋转屏幕后保持Presenter单例问题，因此在new Presenter的时候就传入了View接口。
2.如果想实现Presenter单例，可以是static这种简单粗暴的方式，也可以是Android Loader机制等等。**

## 项目感受
### 1.缺点
1.额外代码的问题还是存在（MVP难免的问题）

### 2.优点
1.将Context也隔离开外，只有在必要的时候再通过View接口获取Context对象。（但实际上这个问题只要网络请求，数据库请求设计妥当，完全可以避免使用Context对象，可以在Application时候初始化即可，之后再也不需要Context对象）
2.规范化View、Presenter，使得项目更加统一，使得编码也更加舒服

# 纠结
文中提及的BasePresenter提供了一个start方法，但是实际上在登陆的时候，是需要根据账号密码登陆的。

所以单纯的无参数的start是满足不了的，所以个人倾向两种方案：

## 1.使用可变参数
```
public void start(String... value);
```
这样就能解决了上面的问题。 这个时候再加上一个验证码校验也是妥妥的，只要我们约定传入的参数是 账号、密码、验证码，就可以了~

但这样的问题也来了，可变参数导致了使用方（Presenter）并不知道里面具体的存放顺序，可能导致一些不可预料的问题。（一不小心密码和账号调换位置，就可能一直登陆失败了！）

所以我想出了下面的方法

## 2.通过View接口返回所需参数
```
public interface ILoginView extends BaseView {
    public String getUserName();//获取账号
    
    public String getUserPsw();//获取密码

    public String getVeriCode();//获取验证码
}
```
这样LoginActivity实现这个接口
```
public class LoginActivity extends BaseActivity implements  ILoginView {
    private LoginActivityPresenter mPresenter;
    private AppCompatEditText mUserName;
    private AppCompatEditText mPsw;
    private AppCompatEditText mCode;
 
   
    public String getUserName(){
            return mUserName.getText().toString();
         }
    
    public String getUserPsw(){
       return mPsw.getText().toString();
    }

    public String getVeriCode(){
      return mCode.getText().toString();
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.login_btn_login:
                mPresenter.start();
                break;
        }
    }
    

    //...
}
```
然后在LoginPresenter执行start的方法中通过View接口获取:
````
public LoginPresenter extends BasePresenter {
        //.....
       @Override
        public void start() {
            String userName  = mView.getUserName();
            String psw = mView.getUserPsw();
            String code = mView.getVeriCode();
            //....执行登陆
       }
}
```

## 3.比较
第二种方法相比第一种，需要写多几个接口，但是这种代价带来的是更好的代码阅读。 所以我个人觉得第二种方案是更佳合适的。

# 后记
实际上，现在的Android架构各有各的说法，有人用Dragger2实现MVVM，有人说MVPVM，也有人还在使用MVC。 

诚然哪一种架构对于Android开发是有利的，但是抛开实际使用来谈优点都是耍流氓，找到自己认为的“完美”架构最重要。 

对于其他架构，我觉得存在即合理，去尝试了解并且为自己所用才是最应该做的。