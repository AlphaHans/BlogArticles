---
title: 【选择图片Jar】Adapter的实现之SparseBooleanArray图片状态管理器（二）
date: 2016-01-29 14:08
---

<h3>1.选择图片遇到的最难解决的问题是？</h3>
本来我想着是直接开始讲解Adapter中的代码，但是考虑到我现在也只是一个大二的学生，接触安卓也不过将近一年。
<!-- more -->
在过去的一年里也遇到了不少的困难，也看了不少的博客和文章，才解决了不少的问题。所以我也想写给像我当初那样的新手看的文章，所以我决定从最原始的开始讲解起来。
回到题目这个问题，我们会遇到最难解决的问题是什么？可能有的朋友已经知道了：当我们重新来回滚动的时候，原来的选中的图片的右上角状态已经消失了。 具体看下面这个几个图：
1.首先我们选中9张图片：
![这里写图片描述](http://img.blog.csdn.net/20160129134003267)
2.然后我们向下滚动一段距离，将上面三图片滚动出屏幕外
![这里写图片描述](http://img.blog.csdn.net/20160129134045005)
3.我们在滚动回来看到
![这里写图片描述](http://img.blog.csdn.net/20160129133749566)

<h3>2.为什么会出现这个问题？</h3>
因为我们将上面三个Item划出屏幕外之后，再重新滚动回来，这个ImageView中的图片重新绘制的！所以，导致我们原来替换的图片和阴影都缺失了

<h3>3.如何解决这个问题？</h3>
解决思路:我们要将选中的item的位置保存到一个数组或者List或者Map之中，然后再Adapter的getView方法中，判断一下这个位置是否被我保存过，然后加载对应状态的图片和阴影。
1.数组：开辟一个与照片数目相当大小的数组，然后再对应位置保存0/1来标识。  这个不现实，如果我有3000张照片，那岂不是要开辟一个3000大小的数组，而很多位置都是没有值的，这样会造成内存的巨大浪费，不值得！
2.List：ArrayList或者LinkedArrayList?这个也不对，因为我不知道这个位置具体对应的状态
3.Map：综上看来Map是我们最好的选择，因为他是键值对的形式，我可以记录我需要的位置的状态。比如我9号对应的是true（表示选中状态）。我们就可以创建一个这样的

```
HaspMap<Integer,Boolean>
```

那这个真的是我们解决的最佳方法吗？性能达到最好了吗？

其实原生控件GridView中也有选中状态和不选中状态的实现方法，即

```
mGridView.setMultiChoiceModeListener(newMultiChoiceModeListener() {.....});
```

来实现。但是好像并没有想象中那么好用，而且用起来比较复杂。不过我们可以参考一下他的思路。
我们查看他的源码

```
  public void setMultiChoiceModeListener(MultiChoiceModeListener listener) {
        if (mMultiChoiceModeCallback == null) {
            mMultiChoiceModeCallback = new MultiChoiceModeWrapper();
        }
        mMultiChoiceModeCallback.setWrapped(listener);
    }

```
再在类中查找mMultiChoiceModeCallback这个，我找到了这个方法

```
public void setItemChecked(int position, boolean value) {
        if (mChoiceMode == CHOICE_MODE_NONE) {
            return;
        }

        // Start selection mode if needed. We don't need to if we're unchecking something.
        if (value && mChoiceMode == CHOICE_MODE_MULTIPLE_MODAL && mChoiceActionMode == null) {
            if (mMultiChoiceModeCallback == null ||
                    !mMultiChoiceModeCallback.hasWrappedCallback()) {
                throw new IllegalStateException("AbsListView: attempted to start selection mode " +
                        "for CHOICE_MODE_MULTIPLE_MODAL but no choice mode callback was " +
                        "supplied. Call setMultiChoiceModeListener to set a callback.");
            }
            mChoiceActionMode = startActionMode(mMultiChoiceModeCallback);
        }

        if (mChoiceMode == CHOICE_MODE_MULTIPLE || mChoiceMode == CHOICE_MODE_MULTIPLE_MODAL) {
            boolean oldValue = mCheckStates.get(position);
            mCheckStates.put(position, value);
            if (mCheckedIdStates != null && mAdapter.hasStableIds()) {
                if (value) {
                    mCheckedIdStates.put(mAdapter.getItemId(position), position);
                } else {
                    mCheckedIdStates.delete(mAdapter.getItemId(position));
                }
            }
            if (oldValue != value) {
                if (value) {
                    mCheckedItemCount++;
                } else {
                    mCheckedItemCount--;
                }
            }
            if (mChoiceActionMode != null) {
                final long id = mAdapter.getItemId(position);
                mMultiChoiceModeCallback.onItemCheckedStateChanged(mChoiceActionMode,
                        position, id, value);
            }
        } else {
            boolean updateIds = mCheckedIdStates != null && mAdapter.hasStableIds();
            // Clear all values if we're checking something, or unchecking the currently
            // selected item
            if (value || isItemChecked(position)) {
                mCheckStates.clear();
                if (updateIds) {
                    mCheckedIdStates.clear();
                }
            }
            // this may end up selecting the value we just cleared but this way
            // we ensure length of mCheckStates is 1, a fact getCheckedItemPosition relies on
            if (value) {
                mCheckStates.put(position, true);
                if (updateIds) {
                    mCheckedIdStates.put(mAdapter.getItemId(position), position);
                }
                mCheckedItemCount = 1;
            } else if (mCheckStates.size() == 0 || !mCheckStates.valueAt(0)) {
                mCheckedItemCount = 0;
            }
        }

        // Do not generate a data change while we are in the layout phase
        if (!mInLayout && !mBlockLayoutRequests) {
            mDataChanged = true;
            rememberSyncState();
            requestLayout();
        }
    }

```
从这个方法名字setItemChecked就可以判断，这个方法就是用来改变选中选中的！ 然后我在凭借着程序员敏锐的嗅觉，在这段代码中找到了这个方法

```
  if (mChoiceMode == CHOICE_MODE_MULTIPLE ||
                    (mChoiceMode == CHOICE_MODE_MULTIPLE_MODAL && mChoiceActionMode != null)) {
    boolean checked = !mCheckStates.get(position, false);
    mCheckStates.put(position, checked);
```

就是你了！我们可以从if语句中看出，这就是多选模式中的实现原理。然后我们看到他保存在了mCheckStates这个东西中。 现在我们来看看你到底是一个什么东西！！

Drung的一下，我们就找了这个东西的真面目啦！！

```
    /**
     * Running state of which positions are currently checked
     */
    SparseBooleanArray mCheckStates;
```

OMG这个是一个什么东西？居然不是我们想象中HashMap? NONONO，凭借着我们奋发向上学习的精神，我查阅了一下官方文档和资料。 原来这个Google针对Android优化的一个HashMap。 哎呀，这一下子可发现了一个不得了的东西啊！！
我们看看这个类的解释：

>  * SparseBooleanArrays map integers to booleans.
 * Unlike a normal array of booleans
 * there can be gaps in the indices.  It is intended to be more memory efficient than using a HashMap to map Integers to Booleans, both because it avoids auto-boxing keys and values and its data structure doesn't rely on an extra entry object for each mapping.
 
现在明白为何Google要出这一个东西了吧？这就要回到Java的知识了，当我们记录状态的时候，我们习惯性put(1,false)；然而HashMap并不是存放这个啊，他会进行自动的装箱，变成Interge和Boolean啊！无论是我们读取还是放入的时候，都会对性能有很大的影响！

SpareBooleanArray就不会出现这种情况，所以他的性能可是高于HashMap。不过要注意的是SpareArray系列的键都只能是int，不能是String，但这也正好满足我们这次的具体需求，因为每一个item本身就有一个position的位置嘛！

<h3>4.回到原来的问题</h3>
回到我们文章一开始的问题。现在小伙伴们应该都知道如何解决一开始那个问题了吧？就是我们可以在Adapter方法getView()中加入一个判断，如果我这个position存入了SpareBooleanArray这里面中，那么我就要做出对应的处理。
比如我是这样

```
@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		.....
		// 处理滑动错位
		handleScrollBackgroundChange(position, viewHolder.show,
				viewHolder.select);
		return convertView;
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
```
这段代码应该很容易就能理解了吧？可能有点问题的是

```
ImageLoaderWrapper.loadFromDrawable(select, selectBtnBackground);
```
这个是什么呢？是UniversalImageLoader的包装类，这个具体的并不是本课程探讨的问题了。只需要知道，这里就是替换了一下背景而已啦~

<h3>5.结束语</h3>
今天的文章就到这里了，有兴趣的可以去了解一下SpareArray全家桶系列。 

下篇文章会带来Adapter的具体封装实现了。本文是为下文铺垫的，要是没有读懂的话还是要好好消化~