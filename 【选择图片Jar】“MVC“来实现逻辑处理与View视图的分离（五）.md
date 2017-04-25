---
title: 【选择图片Jar】“MVC“来实现逻辑处理与View视图的分离（五）
date: 2016-01-29 16:17
---
<h3>1."MVC"是一个什么鬼？</h3>
全称是Controller和View加上一个数据源Model(Data)的简称。
<!-- more -->
本人倾向将Adapter作为M，虽然他也涉及了View，但我觉得实际上Adapter职责是将简单的List数据按照View的模型，组装成方便使用的数据源。所以我将Adapter作为M，有兴趣的可以查阅一下其他资料~。

我们都知道目前Android开发逐渐成熟，出现了很多原来没有的App架构模式， MVC MVP MVVM模式。这都是为了解决App越来越大，导致Activity日益庞大的方法。

这也导致Activity违背了类单一职责（视图展示，逻辑处理）。幸运的是，国内外的高级开发者已经注意到了这个问题，现在的MVC、MVP等项目开发模式将成为未来的开发主流。

本次Jar也是通过这样的模式，来实现逻辑处理和View的分离。

所以本人也希望尽可能往规范化靠拢，所以进行了逻辑和View视图的分离。

当然！引入了Controller这也是为了能够更加方便使用这个Jar包！

<h3>2.先分析一下View的代码</h3>
首先我新建了一个类GalleryGridView，继承自GridView

我们来看看这类类中的方法：
    
    public void bindController//绑定Controller
 
    public void unbindController//解绑Controller
    
    public void bindAdapter//绑定数据源的Adapter
    
	public void onItemClick//设置Item的点击事件的逻辑处理
    
	public GridView getGridView//返回这个GridView实例

现在来看看bindController和unbindController两个具体代码：

```
public void bindController(GalleryViewController controller) {
	mController = controller;
	mController.setPhotoViewListener(this);
}

public void unbindController(GalleryViewController controller) {
	mController.setPhotoViewListener(null);
	mController = null;
}
``` 

这里出现了一个setPhotoViewListener是什么呢？

实际上我这个GallerGridView类还实现了一个接口，这个接口是让Controller来和View通信的桥梁，接口代码如下：


```
public interface IPhotoGridView {

	void bindAdapter(BaseAdapter adapter);

	GridView getGridView();
}
```

相信大家已经理解了吧？ 不理解的话可以去看看MVC或者MVP的文章。这里不展开来讲。

OK!我们现在已经能够了解bindController和unbindController的方法的作用了吧？实际上就是搭建C和V之间的控制桥梁！ 

那现在我们来看看这两个方法：

```
@Override
public void bindAdapter(BaseAdapter adapter) {
	if (adapter != null)
		this.setAdapter(adapter);
}
	
@Override
public GridView getGridView() {
	return this;
}
```

也很好理解吧！

 bindAdapter就是为V设置数据源M！

getGridView显而易见是为了返回当前实例，这个有什么用？
还记得我们在
[【选择图片Jar】拨云见日-GalleryAdapter完整代码（四）](http://blog.csdn.net/qq_18402085/article/details/50607597)
的GalleryAdapter中的getView方法为两个ImageView的setTag方法吗？


```
//设置ShowImageView的tag
viewHolder.show.setTag(path);

//设置SelectImageView的tag
viewHolder.select.setTag(path + "select");
```

到时候我们就可以通过getGridView()返回的实例，通过findViewWithTag找到我们的ImageView,来进行点击事件的逻辑处理哦~！！具体的处理会在下面讲到！

OK！现在还剩下最后一个方法：onItemClick。这个名字是不是看起来好熟悉？没错了，就是设置了item单击事件的回调方法！ 

我们的GalleryGridView还实现了OnItemClickListener的接口，我们重写这个方法，**将点击事件传送到Controller**，进行**逻辑的处理**！！ 这里很重要！代码如下：

```
public void onItemClick(AdapterView<> parent,View view, int position, long id) {
	mController.click(position);
}
```

虽然很简单，但可是包含大学问！

还需要**注意**一点，记得在构造方法中设置监听：

```
public GalleryGridView(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
		//设置item点击事件
		this.setOnItemClickListener(this);
}
```

<h3>3.结束语</h3>
OMG！怎么这么快结束了？说好的C呢？C去哪里了！ 

不要着急，我想看到这里有的同学可能还没有完全明白这样处理的好处，以及MVC的模式的好处在哪里。 

先等等~，不太明白的可以先去查查MVC相关资料哦~。

下篇文章就要带来Controller的具体代码分析了！！
