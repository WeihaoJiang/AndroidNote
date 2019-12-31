---
title: View学习总结
date: 2019-07-16
tags: [android]
categories: android
---


## 背景以及夙愿

- 1 写了一些自定义view的需求，希望有个总结，以供日后查阅
- 2 view不仅仅是自定义view，希望有个全面了解
- 3 养成研究源码的习惯

## View概述

###  官方文档的介绍
View is the base class for widgets, which are used to create interactive UI components (buttons, text fields, etc.). The android.view.ViewGroup subclass is the base class for layouts, which are invisible containers that hold other Views (or other ViewGroups) and define their layout properties.
意思是说：View是窗口小部件的基类，用于创建交互式UI（Button,TextView等都是它的子类），而android.view.ViewGroup子类是布局的基类，它是包含其他视图（或其他ViewGroups）并定义其布局属性的不可见容器。
总结来说：我们看到的所有可视化UI组件都可以看做是View,下面从以下几个方面来了解它
        

## View的滑动
### srollTo和srollBy
#### 看看源码
 
 ```
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
 ```
#### 具体说明
 srollTo()方法：在当前视图内容偏移至(x , y)坐标处，即显示(可视)区域位于(x , y)坐标处。
 srollBy()方法：在当前视图内容继续偏移(x , y)个单位，显示(可视)区域也跟着偏移(x,y)个单位。
 srollTo()和srollBy()移动得都是view的内容，view本身并未移动

#### 举个例子

 下面写了一个demo验证一下：
```
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bn_scrollTo = findViewById(R.id.bn_scrollTo);
        bn_scrollBy = findViewById(R.id.bn_scrollBy);
        text = findViewById(R.id.tv_text);

        bn_scrollTo.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.scrollTo(50,50);
            }
        });
        bn_scrollBy.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.scrollBy(50,50);
            }
        });
        
    }

```
运行效果
![图片](https://s2.ax1x.com/2019/07/16/ZbpygP.gif)
### 通过动画实现
#### 看例子
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bn_scrollTo = findViewById(R.id.bn_scrollTo);
        bn_scrollBy = findViewById(R.id.bn_scrollBy);
        text = findViewById(R.id.tv_text);

        bn_scrollTo.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //v.scrollTo(50,50);
                ObjectAnimator.ofFloat(bn_scrollTo,"translationX",0,200,0,0).setDuration(1000).start();

            }
        });
        bn_scrollBy.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //v.scrollBy(50,50);
                ObjectAnimator.ofFloat(bn_scrollBy,"translationX",0,-200,0,0).setDuration(1000).start();
            }
        });
    }
}
```
运行效果
![图片](https://s2.ax1x.com/2019/07/16/ZbZOQH.gif)
#### 方法说明
这里使用了属性动画ObjectAnimator来实现滑动，个人理解动画并不属于滑动，只是实现了类似滑动的效果，我们可以通过调整动画持续时间setDuration()来做出具有弹性的滑动效果，这里不再多说，关于ObjectAnimator的其他方法可以[参考](https://blog.csdn.net/harvic880925/article/details/50598322)
### 改变LayoutParams
```
   ViewGroup.MarginLayoutParams params = (ViewGroup.MarginLayoutParams) text.getLayoutParams();
        params.width += 100;
        params.leftMargin += 100;
        text.requestLayout();
```
### Scroller

先来看看官方给出的使用样例
```
private Scroller mScroller = new Scroller(context);
  ...
  public void zoomIn() {
      // Revert any animation currently in progress
      mScroller.forceFinished(true);
      // Start scrolling by providing a starting point and
      // the distance to travel
      mScroller.startScroll(0, 0, 100, 0);
      // Invalidate to request a redraw
      invalidate();
  }
```
使用起来非常简单，我们一个一个方法往下分析
先来看看startScroll()
```
  public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }
```
startScroll()方法中保存了滑动起点，滑动距离，滑动时间等等这种参数，但是未见具体滑动代码，真正使view产生滑动效果的是invalidate()
invalidate()方法会导致view重绘，draw方法中调用computeScroll(),computeScroll()是一个空方法需要自己重写
```
   @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            invalidate();
        }
    }
```
向Scroller获取scrollX，scrollY,通过scrollTo()进行滑动，然后再调用invalidate(),重复上述步骤，循环调用直到滑动结束，至于循环多少次呢，是由computeScrollOffset()决定的。
下面仅贴出关键代码
```
  /**
     * Call this when you want to know the new location.  If it returns true,
     * the animation is not yet finished.
     */ 
    public boolean computeScrollOffset() {
      ...

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    
        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE:
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
           ...
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
```
可以看到是通过当前时间和滑动开始时间的差值所占滑动时间的份数来计算的，返回false则滑动结束，否则继续循环。

## View的事件分发机制
这一节更多的会提炼成文字，源码的分析过程涉及子view，父view各自的事件分发三件套方法，不太容易形成书面语言
1、一个view的事件最先传递给它所在的activity的decorview,decorview是当前页面的顶层容器，我们setContentView(R.layout.xxx)设置的view就是decorview的子view
2、然后首先调用顶级View的dispathTouchEvent(),在这个方法中通过onInterceptTouchEvent()来判断是否拦截，如果返回true则由顶级view处理，如果返回false则需要判断是否设置了mOnTouchListener,是则调用onTouch(),否则调用onTouchEvent()
3、子view继续重复1，2步骤分发





