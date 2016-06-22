---
layout: post
title: "Retrofit + EventBus��һ���ܽ�"
date: 2016-06-22 23:42:06 +0800
comments: true
tags: Android
description: Retrofit + EventBus
comments: true
toc: true
---

˵��AndroidӦ�ÿ��������������ܣ�������Ҳ�������������һ����Volley����һ����Retrofit. ����������������Ŀ�ж�Retrofit��Ӧ��ʵ��.
### Retrofit����
Retrofit�����ǳ���Ա������[Jake Wharton](JakeWharton)���ڵ�[Square](https://github.com/square)��˾��Դ��һ��RESTful���������ܣ� Ŀǰ���°���2.1.0������**���Ļ���Retrofit 1.9.x**��2.x �汾�����1.x �зǳ���ĸ��£��Ժ��л������о�һ��2.x�汾. ���Ĳ��漰Retrofit�����Ž��ܣ����˽�����ǿ��Բ鿴[�ĵ�](http://square.github.io/retrofit/)��[Github��Ŀ��ҳ](https://github.com/square/retrofit)�� ������Ҳ�д��������Ž�������. ������Ҫ��������ʵ����Ŀ�ж�Retrofit��EventBus���ʹ�õ��ܽᡣ
### EventBus����
EventBus��һ��Androidƽ̨���¼����߿�ܣ�ʹ�ü򵥡��������Ϳ������������ڴ���Ľ���. �������ܺ��÷���[��Ŀ��ҳ](https://github.com/greenrobot/EventBus). 
### Ϊʲôʹ��EventBus
���֪��Retrofit������һ��API�ӿڵķ�ʽ���£�
���Ƕ���һ��interface ���� `MyRetrofitService`����֪��`MyXxx`�����������е���, ������Ϊʾ��, ��������������һ�°�-_- ��, ��������һ����¼������
```
@Headers("Content-Type: application/json;charset=UTF-8")
@POST("/api/appLogin")
void login(@Body LoginReq loginReqBody, Callback<LoginResp> cb);
```
����`Callback<LoginResp>`��ʹ��Retrofit�ṩ�Ľӿ�`retrofit.Callback<T>`�����ڽ���������Ӧ.  `LoginResp`��`BaseResp`������.

���������UI������е��ýӿڵ�ʱ�������������д����
```Java
MyRestService.getInstance().login(new loginReqBody("username","password"), new Callback<LoginResp> {
    @Override
    public void failure(RetrofitError err) {
        handleRetrofitError(err);
    }

    @Override
    public void success(final LoginResp arg0, Response arg1) {
        //TODO: ������Ӧ
    }
});
```
��ô����ȻUI���ҵ���߼������Retrofit�Ĵ�������������һ�𣬼����ϣ�����. ���������Ŀ���ٰ�Retrofit�ˣ���Ҫ�������������ܣ�����ʹ��Retrofit����ô���еĵ�������ӿڵ�UI����붼Ҫһ���Ķ�����������ά���ɱ��ߣ����Ҽ�������bug.

����Ҫ�����ǽ�ҵ���߼��������������������룬����.

- **����ĳ���**��
����һ��`ResponseHandler`�����࣬ʵ��`Callback<T>`�ӿ�, ��UI�����ʱ����`ResponseHandler`���ʵ��������UI����벻��ֱ������Retrofit�Ĵ��룬��Ϊ����`ResponseHandler`��. `ResponseHandler`���ʵ�ִ������£�
```Java
public abstract class ResponseHandler implements Callback<BaseResp > {

    @Override
    public void success(BaseResp resp, Response response) {
        //TODO: �����Ҫ�Ļ�����һЩ���resp�Ĵ���
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
�����Ѿ��ﵽ�˽����Ŀ�ģ��ñ����һ�������ģ�ÿ���ĵÿ��Ŀ��ļ��ˣ������㲻֪���Ǳ��ֻ�����Ժ���ǲ����Ѿ��������ˣ�����������˵����һ��. �����ֱ�����ģ�����һ�����磬���TA����������ˡ�

- **����һ��**
��������һ���������һ��Activity��������Fragment������б�FragmentA���Ҳ�����FragmentB����FragmentA�е��һ����ťʱ������һ���ӿڣ�Ȼ����ݽӿڷ��ؽ��FragmentB��Ҫ��Ӧ��ˢ�½���.

    ʵ����������кܶ෽ʽ��ʹ���¼������ǱȽϺõ�һ�֡��¼����߿����Լ�ʵ�֣�Ҳ����ʹ��һЩ����Ŀ�Դ��ܣ�����EventBus.
һ�ַ�ʽ����FragmentA�н��սӿڻص���Ȼ������EventBus֪ͨFragmentB. ��ô��Ȼ�Ѿ��������¼����ߣ��β�������ӿ��������Ӧ��EventBus���¼���ʽ���������������뿴�������������Щ�������������ַ�ʽ��ʵ���ܽ�.

### Retrofit + EventBus
**һ**��
����Ҫ��һ��ȫ�ֵ�EventBus����ʵ�������Է���`Application`�Ҳ��������:
```Java
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
**��**��
������Ҫ�������������Activity���̳���һ��`BaseActivity`����`BaseActivity`���һ��`onEvent()`��������Ϊ`onEvent()`������EventBus�ļ�����������е�һ�������������������е�activity��ȥд`onEvent()`����.

��`BaseActivity`��`onCreate()`������ע�����`EventBusProvider.getEventBus().register(this);`
��`onDestroy()`���ע��`EventBusProvider.getEventBus().unregister(this);`

`BaseActivity`�ࣺ
```Java
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

	//���������final�ģ���Ϊ���಻��Ҫ���������������������᷵�ص�ǰ���������
    protected final Type getNwEventMainType() {
        return this.getClass();
    }

    /**
     * NOTE: ��Ҫ��������������Ӧ�����࣬Ҫ�������������������Ӧ
     */
    protected void handleNwEvent(NwEvent event) {
    }
}
```
**NOTE: ** `BaseFragment`��`BaseActivity`����.

`NwEvent`�������¼��ࣺ
```
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

`NwEventType`���¼������࣬`mainType`��ʾ���ĸ��෢�����������Ӧ�¼���`subType`��������һ���෢���Ķ������
```
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

**��**��
��дһ��ResponseHandler�࣬����������Ӧ�ص�����post�¼�����Ҫ���£�
```
class ResponseHandler implements Callback<BaseResp> {
    private NwEventType eventType = null;

	public ResponseHandler(NwEventType eventType) {
		this.eventType = eventType;
	}
	
	@Override
	public void success(BaseResp resp, Response response) {
		//TODO: һЩ���resp���жϡ�����
		ErrorType errorType = ErrorType.OK;//TOOD: ��������߼�.
		boolean result = resp != null;//���������ж�
		
		//Ȼ�󷢳�Event, �ھ�����ýӿڵ�Activity��Fragment�����յ�onEvent()�ص���
		EventBusProvider.getEventBus().post(new NwEvent(eventType, result, resp, errorType));
	}
	
	@Override
	public void failure(RetrofitError error) {
	    //TODO: һЩ���error���жϡ�����
		EventBusProvider.getEventBus().post(new NwEvent(eventType, false, null, errorType));
	}
}
```

**��**��
�ӿ���������һ����
```
@Headers("Content-Type: application/json;charset=UTF-8")
@POST("/api/appLogin")
void login(@Body LoginReq loginReqBody, Callback<LoginResp> cb);
```
Ȼ����`MyRestService`�����Ľӿڸ�Ϊ����һ���¼�����`NwEventType`���ɣ���Ϊ`NwEventType`�е�`mainType`��`subType`�Ѿ��ܹ�ȷ�����ĸ��෢�����ĸ�����
```
public void login(String username, String password, NwEventType eventType) {
	mApiService.login(new LoginReq(username, password), new ResponseHandler(eventType));
}
```

**��**��
UI��ĵ���. ������һ��`Activity`����ýӿڣ�
```
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
		//TODO �����¼��Ӧ
		
		//�����¼�ɹ�����������Ҫ�����û�ȡ�û���Ϣ�ӿ�
		MyRestService.getInstance().getUserInfo(new NwEventType(getNwEventMainType(), NW_EVENT_SUB_TYPE_GET_USER_INFO));
	}
	
	...
}
```

### �ܽ�
���Ϸ�ʽ����ʵ����������UI��ҵ���߼���Ľ���. ���Ƕ���EventBus��ʹ�ò���������ˣ�����NwEventType��ʵ�־Ϳ��Ը�Ϊ��API�ӿڵı����ʵ�֣�����һ���෢��������Ͳ���������һ��Ҫ�Լ����������������Ҳ�Ϳ���ֱ��ʵ��������**"����һ��"**С�����ᵽ������. �ȵ�.

�׻���ש������-_-.