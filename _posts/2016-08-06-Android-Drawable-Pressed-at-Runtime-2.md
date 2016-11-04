---
layout: post
title: "Android Drawable Pressed at Runtime - part 2"
date: 2016-08-09 01:18:06 +0800
comments: true
tags: [Android]
description: Android 按钮 pressed 状态的显示时机 (附少许源码分析)
comments: true
toc: true
---

写这篇文章是想接着上一篇文章 [Android Drawable Pressed at Runtime - part 1](http://tangni.me/2016/08/Android-Drawable-Pressed-at-Runtime-1) ，继续聊聊关于按钮的按下状态的问题。

上一篇文章中，关于不规则形状图片按钮在运行时设置pressed状态的问题，提出了一种思路：用 `PorterDuff.Mode` 的 `SRC_IN` 模式裁剪一张半透明灰度蒙层图来生成与按钮的normal状态图形状一致的蒙层，然后，再叠加两张图生成pressed状态下的图，最后组合成 `StateListDrawable` 来解决 pressed state 的问题。

文章发出后，有评论提示说这种方式“曲线救国”，并提示了可以用着色的方式来简单实现需求，具体可以见他的评论。其实道理很简单，就是用 `ColorFilter` 对图片进行着色，产生pressed的效果。但是着色的方式，相对于 `StateListDrawable` （或selector）的方式，需要自己监听touch事件并判断动作类型来进行着色操作。感谢评论的提示，我具体实现了一下，把实现中遇到的着色时机的细节问题，在这篇文章中记录一下。


### 什么时候显示pressed状态？
由于我们的页面是一个可以滑动并且可以下拉刷新的页面，因此，其页面上的按钮在处理touch事件时就需要考虑区分点击事件和滑动事件。具体到这里说的按钮的点击状态的问题，我们给按钮设置了 `OnTouchListener` ，肯定不能在收到 `ACTION_DOWN` 事件后立刻就设置着色（即设置为pressed状态），因为此时用户手刚接触屏幕，接下来可能是短点击，也可能是滑动，所以需要有一个延时来判断具体是哪种动作，这个延时的时长，系统有一个特定的值，可以通过 `ViewConfiguration` 获取。

![ViewConfiguration.java 源码截图](http://upload-images.jianshu.io/upload_images/71249-df155f102301cd49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

跟进查看 ```TAP_TIMEOUT``` 这个值是100（毫秒），注释说明，如果在这个时长之内用户没有移动，就判定为一次点击，否则判定为scroll。

所以，在我们的实现中，也应该用这个值。在 `OnTouchListener` 的 `onTouch( )` 方法中：

```java
switch (event.getAction()) {
    case MotionEvent.ACTION_DOWN:
        downX = event.getX();
        downY = event.getY();

        // 发送延时消息，进行着色（即设置为pressed状态）
        handler.sendEmptyMessageDelayed(MSG_TINT, ViewConfiguration.getTapTimeout());
        break;

        //...
}
```

如果用户在这个时间段内有移动，则要取消这个消息。如何判定为移动，`ViewConfiguration` 也有个可以获取 `touchSlop` 值的方法，大于这个touchSlop的，则判定为滑动：


![ViewConfiguration.java 源码截图](http://upload-images.jianshu.io/upload_images/71249-7debe8f6ba0fdca6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以，在收到ACTION_MOVE事件时，按这个标准判定是否移动：

```java
//...

case MotionEvent.ACTION_MOVE:
    float dx = event.getX() - downX;
    float dy = event.getY() - downY;
    if (touchSlop == 0) {
        touchSlop = ViewConfiguration.get(target.getContext()).getScaledTouchSlop();
    }

    // 如果判定为移动，在handler中remove掉进行着色的消息
    if ((dx * dx) + (dy * dy) > (touchSlop * touchSlop) ) {
        handler.removeMessages(MSG_TINT);
    }
    break;

case MotionEvent.ACTION_UP:
case MotionEvent.ACTION_CANCEL:
    // 动作结束时，清除着色，按钮由pressed状态恢复为normal状态
    clearTint();
    handler.removeMessages(MSG_TINT);
    break;

//...

```

### 还有一个小问题
实现到这一步，还有个问题：当点击稍微快一些的时候，经常是看不到按钮的pressed状态，即着色后的效果的。原因稍一想也很明显：前面讲到，为了区分滑动和点击，我们并没有在 `ACTION_DOWN` 的时候立刻着色，而是有一个延时，那么如果点击的时候从 `ACTION_DOWN` 到 `ACTION_UP` 的时间小于这个延时，就没有触发着色。

怎么解决？测试发现用 `StateListDrawable` （即selector）的方式是没问题的，点击再快也有pressed效果。而且既然这个时延是从系统获得，那么我们不妨看看源码中是怎么解决这个问题的。开始我猜测源码应该是在ACTION_UP时候做了一次置为pressed状态的动作，然后一定短时间后再取消状态。这样视觉上可以达到效果。`StateListDrawable` （即selector）的方式，依赖于把View ```setPressed(true)```， 那我们就搜索这个方法的调用。查看 `TextView` 源码未发现，再到 `View` 源码，发现果然是这样的：

![View.java 源码截图](http://upload-images.jianshu.io/upload_images/71249-c9f87a49b53c2dd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注释说的很清楚：***按钮在还没来得及显示为pressed状态之前就被release了，那么就现在（在触发click之前）再来显示为pressed状态，确保用户看到效果。***

再往下一点点，还是在这个ACTION_UP的处理中，有取消pressed状态的代码：

![View.java 源码截图](http://upload-images.jianshu.io/upload_images/71249-9b296672b6d93f59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中 ```ViewConfiguration.getPressedStateDuration()``` 就是表示应该显示多久的pressed状态。

这样就全部明朗了，我们也按照这种方式处理即可。文章最后贴上完整代码。

#### 扯远一点
注意上面View源码片段中的 ```if (prepressed)```，这个prepressed 局部变量是怎么回事。我们看它的赋值：

```java
boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
``` 

可以猜测这个prepressed就表示上面提到的没来的及显示pressed状态就ACTION_UP了，需要“补救”。在搜索 ```PFLAG_PREPRESSED``` 这个常量的用处，发现只有一个地方它被赋给了 ```mPrivateFlags``` ：

![View.java 源码截图](http://upload-images.jianshu.io/upload_images/71249-cdc3970f3c27fb55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这段代码就是说在 `ACTION_DOWN` 的时候，判断一下，当前控件如果是一个可滚动布局的子View，那么就延迟设置pressed状态；否则直接设置 ```setPressed(true, x, y)``` 。

看一下 ```isInScrollingContainer``` ：

![View.java 源码截图](http://upload-images.jianshu.io/upload_images/71249-44c3d3ef5e2bb236.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就是沿着View的层级结构一层层往上找，如果有一层的父布局是可滚动的，那么就 `return true`；如果所有层级的父布局都不可滚动，才 `return false`。

再看看 ```ViewGroup``` 中 ```shouldDelayChildPressedState( )``` 这个方法在各个子类的覆写。发现像 ```LinearLayout``` 
、```FrameLayout``` 这样的不可滚动的Layout，返回 false；而 ```ScrollView``` 这样可滚动的则返回 true。

而其实在我的需求中，我们的页面是已知可以滚动的，所以，如果只考虑在这种场景下使用的话，可以省略这一判断。

下面是我写的一个 `PressTintedDrawableViewWrapper` 类，可以支持为 `ImageView` 或 `TextView` 里的Drawable设置pressed效果，我们的按钮是用 `TextView` 的CompoundDrawable实现的，因此用法如下：

```java
new PressTintedDrawableViewWrapper(Color.parseColor("#4C333333")).wrap(someTextView).apply();
```


附上 `PressTintedDrawableViewWrapper` 类完整代码：

```java
public class PressTintedDrawableViewWrapper implements View.OnTouchListener {

    private static final int MSG_TINT = 1;
    private static final long TAP_TIMEOUT = ViewConfiguration.getTapTimeout();

    private View target;
    private Drawable[] drawables;
    private int tintColor;

    private Handler handler;
    private float downX, downY;
    private int touchSlop;

    private boolean tinted = false;

    public PressTintedDrawableViewWrapper(int tintColor) {
        this.tintColor = tintColor;
    }

    public PressTintedDrawableViewWrapper wrap(TextView textView) {
        this.target = textView;
        this.drawables = textView.getCompoundDrawables();
        return this;
    }

    public PressTintedDrawableViewWrapper wrap(ImageView imageView) {
        this.target = imageView;
        this.drawables = new Drawable[]{imageView.getDrawable()};
        return this;
    }

    public boolean apply() {
        if (drawables != null && drawables.length > 0) {
            handler = new TouchHandler(this);
            target.setOnTouchListener(this);

            return true;
        } else {
            return false;
        }
    }

    @Override
    public boolean onTouch(View v, MotionEvent event) {

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                downX = event.getX();
                downY = event.getY();

                handler.sendEmptyMessageDelayed(MSG_TINT, TAP_TIMEOUT);
                break;
            case MotionEvent.ACTION_MOVE:
                float dx = event.getX() - downX;
                float dy = event.getY() - downY;
                if (touchSlop == 0) {
                    touchSlop = ViewConfiguration.get(target.getContext()).getScaledTouchSlop();
                }
                if ((dx * dx) + (dy * dy) > (touchSlop * touchSlop) ) {
                    handler.removeMessages(MSG_TINT);
                }
                break;
            case MotionEvent.ACTION_UP:
                if (!tinted) {
                    if (handler.hasMessages(MSG_TINT)) {
                        handler.removeMessages(MSG_TINT);
                        applyTint();
                        target.postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                clearTint();
                            }
                        }, ViewConfiguration.getPressedStateDuration());
                    }
                } else {
                    clearTint();
                }
                break;
            case MotionEvent.ACTION_CANCEL:
                clearTint();
                handler.removeMessages(MSG_TINT);
                break;
        }

        return false;
    }

    private void applyTint() {
        ColorFilter colorFilter = new PorterDuffColorFilter(tintColor, PorterDuff.Mode.SRC_ATOP);
        for (Drawable drawable : drawables) {
            if (drawable != null) {
                drawable.mutate().setColorFilter(colorFilter);
            }
        }
        tinted = true;
    }

    private void clearTint() {
        if (tinted) {
            for (Drawable drawable : drawables) {
                if (drawable != null) {
                    drawable.mutate().clearColorFilter();
                }
            }
            tinted = false;
        }
    }

    private static class TouchHandler extends Handler {
        WeakReference<PressTintedDrawableViewWrapper> ref;

        public TouchHandler(PressTintedDrawableViewWrapper view) {
            this.ref = new WeakReference<>(view);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_TINT:
                    PressTintedDrawableViewWrapper view = ref.get();
                    if (view != null) {
                        view.applyTint();
                    }
                    break;
            }
        }
    }

}
```