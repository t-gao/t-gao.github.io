---
layout: post
title: "Android Drawable Pressed at Runtime - part 1"
date: 2016-08-06 00:12:06 +0800
comments: true
tags: [Android]
description: Android 运行时给动态加载的图标按钮添加点击效果
comments: true
toc: true
---

大家都知道，要在Android中添加一个带图标的按钮，一般是声明一个ImageView，设置clickable=true，然后设置src为"@drawable/xxx_selector"。或者是声明一个TextView，然后设置它的compoundDrawables为一个xxx_selector。其中xxx_selector是一个在xml中定义的图片选择器，里边有各种状态，其中最多用到的就是 ```android:state_pressed="true"``` 这个状态。本文就只以这个按下状态为例，其他状态同理。一个普通的selector如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/xxx_button_pressed"
          android:state_pressed="true" />
    <item android:drawable="@drawable/xxx_button_normal" />
</selector>
```

这没问题，很简单。但是如果需求是按钮的图标资源不能在客户端编译时写死，而是要在运行时动态去服务器获取，而且如果获取来的图**形状是不确定的**，这种情况下应该怎么添加按钮的按下态，也就是它的点击效果呢？

本文结合我们项目中的实践讲一种思路，供参考，如有更好的方案请观众们赐教。

### 一、固定形状图标

我们APP的主页面顶部类似美团外卖，是几排不固定数量的图标，表示应用的各个功能入口。第一版时，产品的需求和UI设计出来后，确定这些图标一定圆形图标，而且近几版内不会变为其他形状。那么一个图标的normal状态和pressed状态大概如下图所示，按下时稍有变暗。下面以微信的图标为例。

![round_normal.png](http://upload-images.jianshu.io/upload_images/71249-4ddddc9819d804d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![round_pressed.png](http://upload-images.jianshu.io/upload_images/71249-53db3ba25fe22b0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这时，完成这个需求就有三种思路：
1. 两种状态的图都从服务器获取
normal和pressed两张图都从服务器动态获取，然后在客户端拼成一个 ```StateListDrawable``` 。 

2. 只从服务器取normal状态的图，客户端动态生成pressed状态的图，然后拼成一个 ```StateListDrawable```。

3. 热更新，动态加载资源包。

第1种思路优点是pressed状态可以随时动态调整；缺点是增加网络操作，增大流量消耗，增大时延，也增大图片加载失败的风险。第2种思路的优缺点正好相反。如果需求是：正常状态下显示一个微信图标，按下之后要变成一个QQ图标（只是举例），那第2种思路显然不能满足。不过产品和设计都确定不会有这种需求（如有可以考虑第3种思路），于是我们采用了第2种方法。第3种是一种涉及热更新、插件化的思路，本文暂不涉及。

下面来看第2种思路的具体实现。

由于图标固定是圆形的，那么我们只需要一张半透明的灰色蒙层图片叠加在normal状态的图片之上就可以生成pressed状态下的图。蒙层如下图所示：

![round_press_mask.png](http://upload-images.jianshu.io/upload_images/71249-8c5e5648679ec1a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即，**从效果上**： ***round_normal.png + round_press_mask.png = round_pressed.png*** 。叠加两张图的具体代码如下：

```java
// overlay bm2 on top of bm1
public static Bitmap overlayBitmaps(Context context, Bitmap bmp1, Bitmap bmp2,
            int drawableWidth, int drawableHeight, Rect destRect) {

    try {
        int maxWidth = Math.max(bmp1.getWidth(), bmp2.getWidth());
        int maxHeight = Math.max(bmp1.getHeight(), bmp2.getHeight());
        maxWidth = Math.max(maxWidth, drawableWidth);
        maxHeight = Math.max(maxHeight, drawableHeight);

        Bitmap bmOverlay;
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.JELLY_BEAN_MR1) {
            bmOverlay = Bitmap.createBitmap(context.getResources().getDisplayMetrics(),
                    maxWidth, maxHeight, bmp1.getConfig());
        } else {
            bmOverlay = Bitmap.createBitmap(maxWidth, maxHeight, bmp1.getConfig());
        }

        Canvas canvas = new Canvas(bmOverlay);
        canvas.drawBitmap(bmp1, null, destRect, null);
        canvas.drawBitmap(bmp2, null, destRect, null);

        return bmOverlay;
    } catch (Exception e) {
        e.printStackTrace();
        return bmp1;
    }
}
```

normal状态的图片资源从服务器下载而来。关于图片下载，我们没有采用时下流行的Picasso、Glide或Fresco等，自己实现了一个轻量的 **ImageLoader**。ImageLoader 的实现与本文无关，就不介绍了，总之，它从一个image url 加载回来一个 **Bitmap**， 并做了缓存的相关工作。

注意上边的 ```overlayBitmaps()``` 方法只是生成了pressed状态下的图，我们还要用normal状态的图一起来生成一个包含normal和pressed两种状态的 ```StateListDrawable``` 。实现很简单，```new``` 一个  ```StateListDrawable``` ，添加各个状态，注意 **setBounds()** 。代码：

```java
public static StateListDrawable makeStateListDrawable(final Context context, Drawable normal, Bitmap pressedMask,
                                                int drawableWidth, int drawableHeight) {
    if (pressedMask == null || normal == null) {
        return null;
    }

    int pressedWidth = Math.max(drawableWidth, Math.max(normal.getIntrinsicWidth(), pressedMask.getWidth()));
    int pressedHeight = Math.max(drawableHeight, Math.max(normal.getIntrinsicHeight(), pressedMask.getHeight()));

    StateListDrawable stateListDrawable = new StateListDrawable();

    normal.setBounds(0, 0, drawableWidth, drawableHeight);
    Bitmap normalBm = ((BitmapDrawable)normal).getBitmap();

    Rect destRect = new Rect(0, 0, pressedWidth, pressedHeight);//normal.copyBounds();
    Bitmap pressedBitmap = overlayBitmaps(context, normalBm, tailoredMask, drawableWidth, drawableHeight, destRect);

    BitmapDrawable pressed = new BitmapDrawable(context.getResources(), pressedBitmap);
    pressed.setBounds(0, 0, pressedWidth, pressedHeight);

    stateListDrawable.addState(new int[] {android.R.attr.state_pressed}, pressed);
    stateListDrawable.addState(new int[] { }, normal);
    stateListDrawable.setBounds(0, 0, drawableWidth, drawableHeight);

    return stateListDrawable;
}
```

调用的时候，第三个参数 **pressedMask** 传入的就是上面的图 **round_press_mask.png** decode出来的Bitmap：

```java
Bitmap pressedMask = BitmapFactory.decodeResource(context.getResources(), R.drawable.round_press_mask);
```

至此，满足需求，一切都很好。巴特，as always，需求是会变的。

### 二、不定形状图标
 
##### 1. 需求思考
新设计稿一出来，一看，原来乖乖的排排坐吃果果的圆圆的图标们不见了，满屏都换成了各有各形状的图标。也不用含泪去质问设计师是否还记得当初执手许下的约定了，还是想想怎么改代码吧。

虽然图标不是确定的形状了（例如下面的图 icon_random_normal.png），按下效果还是一样，还是稍稍变暗的效果。但是上面的方法就不行了，如果不改，就会变成一个不规则形状的按钮按下之后上面蒙了一层圆形的灰色半透明蒙层，应该会很难看。我们需要根据图标的形状，对应地动态生成一个一样形状的蒙层，然后再套用上面的方法，就可以达到效果。就是说，如果按钮A是三角形的，那么就要生成一个三角形的蒙层，然后叠加生成一个pressed状态的图；如果按钮B是任何不规则形状，同理。


![icon_random_normal.png](http://upload-images.jianshu.io/upload_images/71249-f9fdfbbe9c4bea77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这时候是不是想，如果能把round_press_mask.png 裁剪成需要的形状就好了（需要保证 round_press_mask.png 尺寸足够大）。

##### 2. PorterDuff.Mode
说到这，应该忽然想起来 **PorterDuff** 这个东西了。PorterDuff这个单词查词典基本查不到，其实是关于图像处理的一篇论文的两个作者Thomas Porter 和 Tom Duff 的名字的合成词。定义了一系列处理图像的方式，感兴趣可以查看[这篇文章](http://ssp.impulsetrain.com/porterduff.html)，当然，如果对学术有兴趣的话，也可以看[原论文](http://keithp.com/~keithp/porterduff/p253-porter.pdf) （在下是不敢看的 -_-）。安卓中源码：

```java
public static enum Mode {
    ADD,
    CLEAR,
    DARKEN,
    DST,
    DST_ATOP,
    DST_IN,
    DST_OUT,
    DST_OVER,
    LIGHTEN,
    MULTIPLY,
    OVERLAY,
    SCREEN,
    SRC,
    SRC_ATOP,
    SRC_IN,
    SRC_OUT,
    SRC_OVER,
    XOR;
}
```

对应的效果如下图（图片来自Google搜索）：

![porter_duff_modes.png](http://upload-images.jianshu.io/upload_images/71249-214658ce0b06f33d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有几种都有“裁剪”的效果，其中 `SRC_IN` 是满足我们需求的，可以从一张大的灰色半透明mask图上裁剪下来一片跟normal图标形状一致的子集。

到这儿，原理就清楚了，只需要在上面所说的方法的基础上增加 **“裁剪”** 这一步即可。

我们修改 ```overlayBitmaps()``` 方法，使之增加裁剪的功能，修改后代码如下：

```java
// 增加 isTailoringMask参数，为 true 时表示是在进行裁剪，false 表示是在进行普通的叠加操作
public static Bitmap overlayBitmaps(Context context, Bitmap bmp1, Bitmap bmp2,
            int drawableWidth, int drawableHeight, Rect destRect, boolean isTailoringMask) {

    try {
        int maxWidth = Math.max(bmp1.getWidth(), bmp2.getWidth());
        int maxHeight = Math.max(bmp1.getHeight(), bmp2.getHeight());
        maxWidth = Math.max(maxWidth, drawableWidth);
        maxHeight = Math.max(maxHeight, drawableHeight);

        Bitmap bmOverlay;
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.JELLY_BEAN_MR1) {
            bmOverlay = Bitmap.createBitmap(context.getResources().getDisplayMetrics(), maxWidth, maxHeight, bmp1.getConfig());
        } else {
            bmOverlay = Bitmap.createBitmap(maxWidth, maxHeight, bmp1.getConfig());
        }

        Canvas canvas = new Canvas(bmOverlay);
        canvas.drawBitmap(bmp1, null, destRect, null);

/*******************************************************************/
        // 这里指定一个paint，并设置 PorterDuff.Mode 为 SRC_IN，已达到裁剪效果
        Paint paint = null;
        if (isTailoringMask) {
            paint = new Paint();
            PorterDuff.Mode mode = PorterDuff.Mode.SRC_IN;
            paint.setXfermode(new PorterDuffXfermode(mode));
        }
        canvas.drawBitmap(bmp2, null, destRect, paint);
/*******************************************************************/

        return bmOverlay;
    } catch (Exception e) {
        e.printStackTrace();
        return bmp1;
    }
}
```

好，给 ```overlayBitmaps()``` 方法增加了一项裁剪技能之后，现在来改一下 ```makeStateListDrawable()``` 方法，改后如下：

```java
public static StateListDrawable makeStateListDrawable(final Context context, Drawable normal, Bitmap pressedMask,
                                                int drawableWidth, int drawableHeight) {
    if (pressedMask == null || normal == null) {
        return null;
    }

    int pressedWidth = Math.max(drawableWidth, Math.max(normal.getIntrinsicWidth(), pressedMask.getWidth()));
    int pressedHeight = Math.max(drawableHeight, Math.max(normal.getIntrinsicHeight(), pressedMask.getHeight()));

    StateListDrawable stateListDrawable = new StateListDrawable();

    normal.setBounds(0, 0, drawableWidth, drawableHeight);
    Bitmap normalBm = ((BitmapDrawable)normal).getBitmap();
    
    Rect destRect = new Rect(0, 0, pressedWidth, pressedHeight);//normal.copyBounds();

/*******************************************************************/
    // 先调用一次overlayBitmaps(), isTailoringMask 传 true，这一步只是裁剪出符合形状的 mask
    Bitmap tailoredMask = overlayBitmaps(context, normalBm, pressedMask, drawableWidth, drawableHeight, destRect, true);
    
    // 再调用一次，isTailoringMask 传 false，这一步是将裁剪好的 mask 叠加到 normal 图上， 生成 pressed 状态的图
    Bitmap pressedBm = overlayBitmaps(context, normalBm, tailoredMask, drawableWidth, drawableHeight, destRect, false);
/*******************************************************************/

    BitmapDrawable pressed = new BitmapDrawable(context.getResources(), pressedBm);
    pressed.setBounds(0, 0, pressedWidth, pressedHeight);

    stateListDrawable.addState(new int[] {android.R.attr.state_pressed}, pressed);
    stateListDrawable.addState(new int[] { }, normal);
    stateListDrawable.setBounds(0, 0, drawableWidth, drawableHeight);

    return stateListDrawable;
}
```