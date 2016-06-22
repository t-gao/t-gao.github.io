---
layout: post
title: "Retrofit + EventBus的一点总结"
date: 2016-06-22 23:42:06 +0800
comments: true
tags: Android
description: Retrofit + EventBus
comments: true
toc: true
---

说起Android应用开发的网络请求框架，最流行也最优秀的两个，一个是Volley，另一个是Retrofit. 今天来聊聊我在项目中对Retrofit的应用实践.
### Retrofit介绍
Retrofit是明星程序员、男神[Jake Wharton](JakeWharton)所在的[Square](https://github.com/square)公司开源的一个RESTful网络请求框架， 目前最新版是2.1.0，但是**本文基于Retrofit 1.9.x**，2.x 版本相对于1.x 有非常大的更新，以后有机会再研究一下2.x版本. 本文不涉及Retrofit的入门介绍，不了解的亲们可以查看[文档](http://square.github.io/retrofit/)和[Github项目主页](https://github.com/square/retrofit)， 网络上也有大量的入门介绍文章. 本文主要讲讲我在实际项目中对Retrofit和EventBus结合使用的总结。
### EventBus介绍
EventBus是一个Android平台的事件总线框架，使用简单、轻量、低开销，可以用于代码的解耦. 基本介绍和用法见[项目主页](https://github.com/greenrobot/EventBus). 
### 为什么使用EventBus
大家知道Retrofit中声明一个API接口的方式如下：
我们定义一个interface 叫做 `MyRetrofitService`（我知道`MyXxx`这样的命名有点土, 但是作为示例, 观众朋友们忍耐一下吧-_- ）, 里面声明一个登录方法：

```java
@Headers("Content-Type: application/json;charset=UTF-8")
@POST("/api/appLogin")
void login(@Body LoginReq loginReqBody, Callback<LoginResp> cb);
```

其中`Callback<LoginResp>`是使用Retrofit提供的接口`retrofit.Callback<T>`，用于接收请求响应.  `LoginResp`是`BaseResp`的子类.

如果我们在UI层代码中调用接口的时候是类似下面的写法：

```java
MyRestService.getInstance().login(new loginReqBody("username","password"), new Callback<LoginResp> {
    @Override
    public void failure(RetrofitError err) {
        handleRetrofitError(err);
    }

    @Override
    public void success(final LoginResp arg0, Response arg1) {
        //TODO: 处理响应
    }
});
```

那么很显然UI层的业务逻辑代码和Retrofit的代码紧紧耦合在了一起，剪不断，理还乱. 如果哪天项目不再爱Retrofit了，想要更换网络请求框架，不再使用Retrofit，那么所有的调用网络接口的UI层代码都要一番改动，这样代码维护成本高，而且极易引入bug.

我们要做的是将业务逻辑和网络请求两层做分离，解耦.

- **最初的尝试**：
定义一个`ResponseHandler`抽象类，实现`Callback<T>`接口, 在UI层调用时传入`ResponseHandler`类的实例，这样UI层代码不再直接依赖Retrofit的代码，改为依赖`ResponseHandler`类. `ResponseHandler`类的实现大致如下：

```java
public abstract class ResponseHandler implements Callback<BaseResp > {

    @Override
    public void success(BaseResp resp, Response response) {
        //TODO: 如果需要的话，做一些针对resp的处理
        onSuccess(resp);
    }

    @Override
    public void failure(RetrofitError retrofitError) {
        int errCode = getErrCode(retrofitError);
        String errMsg = getErrMsg(retrofitError);
        onFailure0(errCode, errMsg);
    }

    public abstract void onSuccess(BaseResp resp);
    public abstract void onFailure0(int errorCode, String errorMsg);
}
```

这样已经达到了解耦的目的，好比你跟一个人网聊，每天聊得开心开心极了，但是你不知道那边手机或电脑后边是不是已经换了人了，反正对你来说体验一样. 相对于直接面聊，隔了一层网络，你和TA被网络解耦了。

- **更进一步**
考虑这样一个需求，如果一个Activity里有两个Fragment，左侧列表FragmentA和右侧详情FragmentB，在FragmentA中点击一个按钮时，调用一个接口，然后根据接口返回结果FragmentB需要相应的刷新界面.

    实现这个需求有很多方式，使用事件总线是比较好的一种。事件总线可以自己实现，也可以使用一些优秀的开源框架，比如EventBus.
一种方式是在FragmentA中接收接口回调，然后再用EventBus通知FragmentB. 那么既然已经引入了事件总线，何不把网络接口请求的响应用EventBus的事件形式发出来，这样代码看起来更清楚美观些，接下来是这种方式的实践总结.

### Retrofit + EventBus
**一**，
首先要有一个全局的EventBus单例实例，可以放在`Application`里，也可以如下:

```java
    public class EventBusProvider {
        private static final EventBus mEventBus;
        static {
            mEventBus = EventBus.builder().logNoSubscriberMessages(false).sendNoSubscriberEvent(false).build();
        }

        public static final EventBus getEventBus() {
            return mEventBus;
        }
}
```

**二**，
所有需要处理网络请求的Activity都继承自一个`BaseActivity`，在`BaseActivity`里加一个`onEvent()`方法，因为`onEvent()`方法是EventBus的监听者类必须有的一个方法，这样避免所有的activity都去写`onEvent()`方法.

在`BaseActivity`的`onCreate()`方法里注册监听`EventBusProvider.getEventBus().register(this);`
在`onDestroy()`里解注册`EventBusProvider.getEventBus().unregister(this);`

`BaseActivity`类：

```java
public class BaseActivity extends Activity {

	@Override
    protected void onCreate(Bundle savedInstanceState) {
	    EventBusProvider.getEventBus().register(this);
        super.onCreate(savedInstanceState);
    }

	@Override
    protected void onDestroy() {
		EventBusProvider.getEventBus().unregister(this);
		super.onDestroy();
	}

	public final void onEvent(NwEvent event) {
        if (event != null && event.type.mainType == getNwEventMainType()) {
            handleNwEvent(event);
        }
    }

	//这个方法是final的，因为子类不需要覆盖这个方法，这个方法会返回当前子类的类型
    protected final Type getNwEventMainType() {
        return this.getClass();
    }

    /**
     * NOTE: 需要处理网络请求响应的子类，要覆盖这个方法来处理响应
     */
    protected void handleNwEvent(NwEvent event) {
    }
}
```

**NOTE: ** `BaseFragment`与`BaseActivity`类似.

`NwEvent`是网络事件类：

```java
public class NwEvent {

	public NwEventType type = null;
	public BaseResp content = null;
	public boolean success = false;
	public MyRestService.ErrorType errorType = MyRestService.ErrorType.NOT_SPECIFIED;

	public NwEvent() {
		this.type = new NwEventType();
	}

	public NwEvent(NwEventType type, boolean success, BaseResp content, MyRestService.ErrorType errorType) {
		this.type = type != null ? type : new NwEventType();
		this.content = content;
		this.success = success;
		this.errorType = errorType;
	}
}
```

`NwEventType`是事件类型类，`mainType`表示是哪个类发出的请求的响应事件，`subType`用于区分一个类发出的多个请求：

```java
public class NwEventType {
	public Type mainType = null;
	public int subType = -1;

	public NwEventType() {
	}

	public NwEventType(Type type) {
		this.mainType = type;
	}

	public NwEventType(Type mainType, int subType) {
		this.mainType = mainType;
		this.subType = subType;
	}

	@Override
	public String toString() {
		return "{ mainType: " + mainType + ", subType: " + subType + " }";
	}
}
```

**三**，
另写一个ResponseHandler类，处理网络响应回调，并post事件，简要如下：

```java
class ResponseHandler implements Callback<BaseResp> {
    private NwEventType eventType = null;

	public ResponseHandler(NwEventType eventType) {
		this.eventType = eventType;
	}
	
	@Override
	public void success(BaseResp resp, Response response) {
		//TODO: 一些针对resp的判断、处理
		ErrorType errorType = ErrorType.OK;//TOOD: 根据你的逻辑.
		boolean result = resp != null;//或更具体的判断
		
		//然后发出Event, 在具体调用接口的Activity或Fragment就能收到onEvent()回调了
		EventBusProvider.getEventBus().post(new NwEvent(eventType, result, resp, errorType));
	}
	
	@Override
	public void failure(RetrofitError error) {
	    //TODO: 一些针对error的判断、处理
		EventBusProvider.getEventBus().post(new NwEvent(eventType, false, null, errorType));
	}
}
```

**四**，
接口声明还是一样：

```java
@Headers("Content-Type: application/json;charset=UTF-8")
@POST("/api/appLogin")
void login(@Body LoginReq loginReqBody, Callback<LoginResp> cb);
```

然后在`MyRestService`里对外的接口改为传入一个事件类型`NwEventType`即可，以为`NwEventType`中的`mainType`和`subType`已经能够确定是哪个类发出的哪个请求：

```java
public void login(String username, String password, NwEventType eventType) {
	mApiService.login(new LoginReq(username, password), new ResponseHandler(eventType));
}
```

**五**，
UI层的调用. 比如在一个`Activity`里调用接口：

```java
class MyExampleActivity extends BaseActivity {

    private static final int NW_EVENT_SUB_TYPE_LOGIN = 1;
	private static final int NW_EVENT_SUB_TYPE_GET_USER_INFO = 2;

	...
	
	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		
		...
		MyRestService.getInstance().login(un, pw, new NwEventType(getNwEventMainType(), NW_EVENT_SUB_TYPE_LOGIN));
		...
		
		
	}

	...
	
	@Override
    protected void handleNwEvent(NwEvent event) {
		switch (event.type.subType) {
			case NW_EVENT_SUB_TYPE_LOGIN:
				handleLoginResp(event);
				break;
			case NW_EVENT_SUB_TYPE_GET_USER_INFO:
			    handleUserInfoResp(event);
			    break;
			
		}
	}
	
	...
	
	private void handleLoginResp(NwEvent event) {
		//TODO 处理登录响应
		
		//如果登录成功，而且有需要，调用获取用户信息接口
		MyRestService.getInstance().getUserInfo(new NwEventType(getNwEventMainType(), NW_EVENT_SUB_TYPE_GET_USER_INFO));
	}
	
	...
}
```

### 总结
以上方式基本实现了网络层和UI、业务逻辑层的解耦. 但是对于EventBus的使用并不限于如此，比如NwEventType的实现就可以改为用API接口的编号来实现，这样一个类发出的请求就不被拘泥于一定要自己这个类来监听处理，也就可以直接实现我们在**"更进一步"**小节中提到的需求. 等等.

献花拍砖请随意-_-.