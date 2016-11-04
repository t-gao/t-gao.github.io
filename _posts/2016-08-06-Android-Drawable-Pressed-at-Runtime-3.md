---
layout: post
title: "Android Drawable Pressed at Runtime - part 3"
date: 2016-08-10 16:34:06 +0800
comments: true
tags: [Android]
description: Android Drawable / DrawableCompat # setTintList( ) 使用时一个值得注意的问题
comments: true
toc: true
---

本文还是接续上两篇文章，继续聊运行时动态处理按钮点击状态的问题，本文是这个问题第三种解决方案，应该也是三个中最合理的方案。这三篇依次看下来，可以看到解决一个问题走过的弯路：
- [Android Drawable Pressed at Runtime - part 1](http://tangni.me/2016/08/Android-Drawable-Pressed-at-Runtime-1)
- [Android Drawable Pressed at Runtime - part 2](http://tangni.me/2016/08/Android-Drawable-Pressed-at-Runtime-2)
- [Android Drawable Pressed at Runtime - part 3 （本文）](http://tangni.me/2016/08/Android-Drawable-Pressed-at-Runtime-3)

也许本文还是弯路（或许再探索一下源码还可以发现更好的方案），不过走弯路也不是完全没有意义，走弯路过程中用到的一些方法也许可以在其他的场景下提供一些启发。

第一篇文章介绍了一种裁剪蒙层图、生成 `StateListDrawable` 的方式；第二篇讲了着色的方式，但是需要自己设置 `OnTouchListener` 来确定着色的时机；本文介绍的方式利用系统View源码中自己的状态转换机制，使用 `DrawableCompat.setTintList()` 来完成各个状态（这里的需求主要是pressed 状态）下的着色。本文主要说一下这种方式在处理pressed状态时一个需要注意的问题。

`Drawable` 类的 `setTintList()` 和 `setTintMode()` 方法都是在API level 21 才引入的方法。所以要兼容低版本需要使用 `DrawableCompat.setTintList()` 这个静态方法。我们这里要处理的是pressed状态，代码大致如下：

```java
Drawable drawable = new BitmapDrawable(context.getResources(), bitmap); // bitmap从服务端加载而来

int[][] colorStates = new int[][] {
		new int[] {android.R.attr.state_pressed},
		new int[] { }
};

int[] colors = new int[]{context.getResources().getColor(pressTintColorId), Color.TRANSPARENT};
ColorStateList colorStateList = new ColorStateList(colorStates, colors);

finalDrawable = DrawableCompat.wrap(drawable);
finalDrawable = finalDrawable.mutate();
DrawableCompat.setTintList(finalDrawable, colorStateList);
DrawableCompat.setTintMode(finalDrawable, PorterDuff.Mode.SRC_ATOP);
```

 
### 问题所在

这段代码跑在API 21 及以上的设备上时，在按下按钮后并没有着色效果。

我们来看一下源码。

这种方式设置了一个 `ColorStateList` ，因此需要View在被按下时在pressed state下使用tint color。查看 `View` 类源码，`setPressed()`  方法调用了 `refreshDrawableState()` 方法， `refreshDrawableState()` 方法里调用了 `drawableStateChanged()`， `drawableStateChanged()` 里 则调用了 Drawable 的 `setState()` 方法。我们来看一些 `setState()` 方法源码：


![Drawable.java 源码片段](http://upload-images.jianshu.io/upload_images/71249-908f7fd8b5e1bc18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看来主要在 `onStateChange()`，它的实现在各个子类里，我们这里只看 `BitmapDrawable`：


![BitmapDrawable.java 源码片段](http://upload-images.jianshu.io/upload_images/71249-fa514a505d37fb89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`updateTintFilter()` 的实现又回到父类 `Drawable` 类：


![Drawable.java 源码片段](http://upload-images.jianshu.io/upload_images/71249-6f5b74497bb3fe75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这个方法只是更新了tintFilter的color和mode，并没有触发Drawable的绘制。所以，从 `setPressed()`  方法跟踪到此，发现都没有触发Drawable重新绘制的代码，所以按下时的着色 tint color自然没有显示出来。

那为什么在 **API 21 以下反而是有效的** 呢？

再看一下 `DrawableCompat.java` 中的 `setTintList()` 的实现，这里直接给到 `DrawableCompat` 的API 21的具体实现类 `DrawableCompatLollipop`：


![DrawableCompatLollipop.java 源码片段](http://upload-images.jianshu.io/upload_images/71249-3e824172a85f894f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，两个方法在API 21 及以上都是直接调用了 `Drawable` 自己的方法 `setTintList()` 和 `setTintMode()` ；而在 API 21 以下是调用了 `DrawableCompatBase` 里的方法。`DrawableCompatBase` 则调用了 `DrawableWrapper` 的方法，`DrawableWrapper` 是一个 interface，它的方法实现在具体类里，类继承/实现层级关系如下：

![DrawableWrapper 的实现类层级](http://upload-images.jianshu.io/upload_images/71249-685ea70d811e93b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要的方法实现都在第一层子类 `DrawableWrapperDonut` 中，。看一下 `DrawableWrapperDonut` 中的 `setState()` 方法的实现：


![DrawableWrapperDonut.java 源码片段](http://upload-images.jianshu.io/upload_images/71249-5e10b032277f8143.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到调用了 `updateTint()` ，就是这个方法触发Drawable的重新绘制，才能看到pressed state下着色的效果：


![DrawableWrapperDonut.java 源码片段](http://upload-images.jianshu.io/upload_images/71249-229f564fd5b6d734.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在，弄清楚了API 21以下有效的原因，就可以解决API 21及以上无效的问题了。我们也在 `setState()` 后触发一下Drawable重绘。

在上面的分析中，调用栈：`View#setPressed()` -> `View#refreshDrawableState()` -> `View#drawableStateChanged()` -> ``Drawable#setState()` -> `BitmapDrawable#onStateChange()` 。

那么我们就新定义一个继承自 `BitmapDrawable` 的子类，覆盖 `onStateChange()` ：

```java
@Override
protected boolean onStateChange(int[] stateSet) {
	boolean ret = super.onStateChange(stateSet);

	if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
		if (ret) {
			// 触发Drawable重新绘制，以使利用setTintList()设置的各个状态下的tint效果得到显示
			invalidateSelf();
		}
	}

	return ret;
}
```

然后最上边 ```Drawable drawable = new BitmapDrawable(context.getResources(), bitmap);``` 改为 new 这个自定义的子类的实例即可：

```java
Drawable drawable = new SomeExtendedBitmapDrawable(context.getResources(), bitmap);
```