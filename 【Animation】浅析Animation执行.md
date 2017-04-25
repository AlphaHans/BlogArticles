---
title: 【Animation】浅析Animation执行
date: 2016.4.18 1.52
tag: Animation
category: Android进阶
---

# View#startAnimation
```
view.startAnimation(animation);
```
<!-- more -->
startAnimation源代码如下：
```
/**
 * Start the specified animation now.
 *
 * @param animation the animation to start now
 */
public void startAnimation(Animation animation) {
    //设置开始时间
    animation.setStartTime(Animation.START_ON_FIRST_FRAME);
    //设置动画
    setAnimation(animation);
    //刷新父类缓存
    invalidateParentCaches();
    //刷新View本身及其子View（备注：这个方法开发中也可以主动调用的~子线程是postInvalidate）
    invalidate(true);
}
```

# View#invalidate
```
void invalidate() {
    invalidate(true);
}
void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}

void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    if (mGhostView != null) {
        mGhostView.invalidate(true);
        return;
    }
    //是否跳过重新绘制
    if (skipInvalidate()) {
        return;
    }
    // 一堆判断
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        // 省略代码

        // Propagate the damage rectangle to the parent view.
        final AttachInfo ai = mAttachInfo;//这个View的信息
        final ViewParent p = mParent;//包含这个View的父容器
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);

            p.invalidateChild(this, damage);//重点
        }

        // Damage the entire projection receiver, if necessary.
        if (mBackground != null && mBackground.isProjected()) {
            final View receiver = getProjectionReceiver();
            if (receiver != null) {
                receiver.damageInParent();
            }
        }

        // Damage the entire IsolatedZVolume receiving this view's shadow.
        if (isHardwareAccelerated() && getZ() != 0) {
            damageShadowReceiver();
        }
    }
}
```
重点：
```
final ViewParent p = mParent;//包含这个View的父容器
//....
p.invalidateChild(this, damage);
```
看看mParent这个是什么属性：
```
/**
 * The parent this view is attached to.
 * {@hide}
 *
 * @see #getParent()
 */
protected ViewParent mParent;
/**
 * Gets the parent of this view. Note that the parent is a
 * ViewParent and not necessarily a View.
 *
 * @return Parent of this view.
 */
public final ViewParent getParent() {
    return mParent;
}
```
我们可以看到getParent方法的注释：获取这个View的父容器。注意，这个parent可以不是一个View。
这句话是什么意思？
首先，在我们这个子View中他的父容器肯定是一个ViewGroup。但有没有想过ViewGroup的父容器是什么呢？
是不是细思极恐呢？ 那么到底是什么呢？其实ViewGroup的父容器是ViewRootImpl。这个我们在后面也会介绍。
# ViewGroup#invalidateChild
```
**ViewGroup**
/**
 * Don't call or override this method. It is used for the implementation of
 * the view hierarchy.
 */
public final void invalidateChild(View child, final Rect dirty) {
    ViewParent parent = this;
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
         //省略...
        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }
            if (drawAnimation) {
                if (view != null) {
                    view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                } else if (parent instanceof ViewRootImpl) {
                    ((ViewRootImpl) parent).mIsAnimating = true;
                }
            }

            // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
            // flag coming from the child that initiated the invalidate
            if (view != null) {
               //省略...
            }

            parent = parent.invalidateChildInParent(location, dirty);
            if (view != null) {
                //省略...
                }
            }
        } while (parent != null);
    }
}
```
我们可以看到
```
ViewParent parent = this;
//....
parent = parent.invalidateChildInParent(location, dirty);
```
ViewParent是什么？它是一个接口类，而且ViewGroup也实现了这个接口，所以他就调用了
```
/**
 * Don't call or override this method. It is used for the implementation of
 * the view hierarchy.
 *
 * This implementation returns null if this ViewGroup does not have a parent,
 * if this ViewGroup is already fully invalidated or if the dirty rectangle
 * does not intersect with this ViewGroup's bounds.
 */
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & PFLAG_DRAWN) == PFLAG_DRAWN ||
            (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID) {
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                    FLAG_OPTIMIZE_INVALIDATE) {
            //....
            return mParent;
        } else {
           //....
            return mParent;
        }
    }

    return null;
}
ViewGroup的invalidateChild函数通过循环不断调用其父视图的invalidateChildInParent。而ViewRoot又是DecorView的父视图。所以，最终ViewRoot的invalidateChildInParent会被调用。
```
# ViewRootImp#invalidateChildInParent
而ViewParent的实现类是ViewRootImp，找到invalidateChild具体实现：
```    
906 
907     @Override
908     public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
909         checkThread();
910         if (DEBUG_DRAW) Log.v(TAG, "Invalidate child: " + dirty);
911 
912         if (dirty == null) {
913             invalidate();
914             return null;
915         } else if (dirty.isEmpty() && !mIsAnimating) {
916             return null;
917         }
918 
919         if (mCurScrollY != 0 || mTranslator != null) {
920             mTempRect.set(dirty);
921             dirty = mTempRect;
922             if (mCurScrollY != 0) {
923                 dirty.offset(0, -mCurScrollY);
924             }
925             if (mTranslator != null) {
926                 mTranslator.translateRectInAppWindowToScreen(dirty);
927             }
928             if (mAttachInfo.mScalingRequired) {
929                 dirty.inset(-1, -1);
930             }
931         }
932 
933         final Rect localDirty = mDirty;
934         if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
935             mAttachInfo.mSetIgnoreDirtyState = true;
936             mAttachInfo.mIgnoreDirtyState = true;
937         }
938 
939         // Add the new dirty rect to the current one
940         localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
941         // Intersect with the bounds of the window to skip
942         // updates that lie outside of the visible region
943         final float appScale = mAttachInfo.mApplicationScale;
944         final boolean intersected = localDirty.intersect(0, 0,
945                 (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
946         if (!intersected) {
947             localDirty.setEmpty();
948         }
949         if (!mWillDrawSoon && (intersected || mIsAnimating)) {
950             scheduleTraversals();
951         }
952 
953         return null;
954     }

```
# ViewRootImpl#scheduleTraversals
关键：
```
949         if (!mWillDrawSoon && (intersected || mIsAnimating)) {
950             scheduleTraversals();
951         }
```

```
1028    void scheduleTraversals() {
1029        if (!mTraversalScheduled) {
1030            mTraversalScheduled = true;
1031            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
1032            mChoreographer.postCallback(
1033                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
1034            if (!mUnbufferedInputDispatch) {
1035                scheduleConsumeBatchedInput();
1036            }
1037            notifyRendererOfFramePending();
1038        }
1039    }
```
# ViewRootImpl#TraversalRunnable
重点是这一句，mTraversalRunnable是执行界面刷新的Runnable任务的对象，mChoreographer里面封装了一个Handler。
```
1032            mChoreographer.postCallback(
1033                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```
mTraversalRunnable源代码下
```
5882    final class TraversalRunnable implements Runnable {
5883        @Override
5884        public void run() {
5885            doTraversal();
5886        }
5887    }
5888    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```
# ViewRootImpl#doTraversal
```
1050    void doTraversal() {
1051        if (mTraversalScheduled) {
1052            //省略多行代码：
1060            try {
1061                performTraversals();//重点
1062            } finally {
1063                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
1064            }
1065
1066            //省略多行代码
1070        }
1071    }
```
# ViewRootImpl#performTraversals
这里的代码超级多，所以就不贴出来了。

最终就会通过这样，调用View的绘制方法了。

# ViewGroup#drawChild
```
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

# View#draw
```
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    //省略代码
    final Animation a = getAnimation();
    if (a != null) {
        //执行动画绘制
        more = drawAnimation(parent, drawingTime, a, scalingRequired);
        //省略
    } else {
        //省略N行
    }

    mRecreateDisplayList = false;

    return more;
}
```
# View#drawAnimation
```
private boolean drawAnimation(ViewGroup parent, long drawingTime,
        Animation a, boolean scalingRequired) {
    Transformation invalidationTransform;
    final int flags = parent.mGroupFlags;
    final boolean initialized = a.isInitialized();//是否已经初始化
    if (!initialized) {//没有初始化就初始化
        a.initialize(mRight - mLeft, mBottom - mTop, parent.getWidth(), parent.getHeight());
        a.initializeInvalidateRegion(0, 0, mRight - mLeft, mBottom - mTop);
        if (mAttachInfo != null) a.setListenerHandler(mAttachInfo.mHandler);
        onAnimationStart();//设置动画监听器，开始绘制动画
    }
    //存储动画信息
    final Transformation t = parent.getChildTransformation();


    //动画开始
    boolean more = a.getTransformation(drawingTime, t, 1f);


    if (scalingRequired && mAttachInfo.mApplicationScale != 1f) {
        //.....
    }

    if (more) {
        //根据实现，判断动画是否需要进行调整，然后刷新不同的区域
        if (!a.willChangeBounds()) {
            if ((flags & (ViewGroup.FLAG_OPTIMIZE_INVALIDATE | ViewGroup.FLAG_ANIMATION_DONE)) ==
                    ViewGroup.FLAG_OPTIMIZE_INVALIDATE) {
                parent.mGroupFlags |= ViewGroup.FLAG_INVALIDATE_REQUIRED;
            } else if ((flags & ViewGroup.FLAG_INVALIDATE_REQUIRED) == 0) {
                // The child need to draw an animation, potentially offscreen, so
                // make sure we do not cancel invalidate requests
                parent.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                parent.invalidate(mLeft, mTop, mRight, mBottom);
            }
        } else {
            if (parent.mInvalidateRegion == null) {
                parent.mInvalidateRegion = new RectF();
            }
            final RectF region = parent.mInvalidateRegion;
           //获取重绘区域
            a.getInvalidateRegion(0, 0, mRight - mLeft, mBottom - mTop, region,
                    invalidationTransform);

            // The child need to draw an animation, potentially offscreen, so
            // make sure we do not cancel invalidate requests
            parent.mPrivateFlags |= PFLAG_DRAW_ANIMATION;

            final int left = mLeft + (int) region.left;
            final int top = mTop + (int) region.top;
            //重绘
            parent.invalidate(left, top, left + (int) (region.width() + .5f),
                    top + (int) (region.height() + .5f));
        }
    }
    return more;
}
```

# Animation#getTransformation
```
public boolean getTransformation(long currentTime, Transformation outTransformation,
        float scale) {
    mScaleFactor = scale;
    return getTransformation(currentTime, outTransformation);
}

public boolean getTransformation(long currentTime, Transformation outTransformation) {
    //...
    float normalizedTime;
    if (duration != 0) {
        normalizedTime = ((float) (currentTime - (mStartTime + startOffset))) /
                (float) duration;
    } else {
        // time is a step-change with a zero duration
        normalizedTime = currentTime < mStartTime ? 0.0f : 1.0f;
    }
    //是否完成
    final boolean expired = normalizedTime >= 1.0f;
    mMore = !expired;

    //....
    if ((normalizedTime >= 0.0f || mFillBefore) && (normalizedTime <= 1.0f || mFillAfter)) {
        //....
        //根据不同的Interpolator获取具体执行的动画百分比
        final float interpolatedTime = mInterpolator.getInterpolation(normalizedTime);
        //开始动画效果
        applyTransformation(interpolatedTime, outTransformation);
    }
    //如果完成
    if (expired) {
       //....
    }

    if (!mMore && mOneMoreTime) {
        mOneMoreTime = false;
        return true;
    }

    return mMore;
}
```

# Animation#applyTransformation
```
protected void applyTransformation(float interpolatedTime, Transformation t) {

}
```
这个是一个空方法，不同的进行的不同具体实现

# 结束
基本一个动画执行的过程就跑了一遍。

当然涉及了View绘制的过程，可能会有一些错误的地方。 如果有请之处哈~~


**-Hans 2016.4.18 1:53**

# 4.28更新
修正了ViewGroup#invalidateChild的调用细节
如果对上述不理解调用过程不理解的，可以看看[ztelur前辈](http://blog.csdn.net/u012422440)的这一张图
![引用ztelur前辈一张图](http://ww3.sinaimg.cn/large/005SiNxyjw1f35dzciapvj30eb0jcdgt.jpg)
感谢：[ztelur前辈](http://blog.csdn.net/u012422440)

**-Hans 2016.4.28 13:24**