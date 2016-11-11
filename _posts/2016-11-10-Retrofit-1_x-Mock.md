---
layout: post
title: "一种基于Retrofit 1.x的简单Mock机制"
date: 2016-11-10 19:44:12 +0800
comments: true
tags: [Retrofit]
description: 通过装饰retrofit.client.Client实现mock，用注解，用法简单
comments: true
toc: true
---

#### 背景
客户端开发过程中难免会遇到服务器接口尚未开发完成、服务器正在部署、测试服务器挂了等等情况导致接口无法访问，影响客户端的调试。如果你们的APP的开发流程是某一个版本的客户端和服务端同步开发的话，这种情况会更频繁。这时候我们就需要一个 Mock service 来为服务器接口请求做一个“假冒”的响应了。

本文介绍我们项目基于 **Retrofit 1.9.0** 的一直简单的 Mock 实现。[Retrofit 2.x](https://square.github.io/retrofit/) 已经出来很长时间了，而且 2.x 有 [內建的 Mock 机制](https://github.com/square/retrofit/blob/master/samples/src/main/java/com/example/retrofit/SimpleMockService.java)，所以对于 2.x，本文就没有直接意义了。俺们为什么没有升级 2.x，因为正如选择一个第三方库是一个需要十分慎重的决定一样，升级一个第三方库也需要慎重，尤其是大版本升级。Retrofit 2.x 相对于1.x 有很大的变化，在俺们的项目目前用 1.9.0 没有出过什么问题的情况下，不想冒险去升级2.x 。所以基于 1.9.0 实现了这很简单的 Mock，只在 Retrofit 1.9.0 测试过，但是相信 Retrofit 1.x 都能使用。

#### 发现突破口
关于 Retrofit 的使用，前面的 [这篇文章](http://tangni.me/2016/06/Retrofit-With-EventBus) 已经有介绍。
Retrofit 1.x 需要先 `new RestAdapter.Builder()`，并且需要调用这个 Builder 实例的 `setClient(final Client client)` 方法来设置 Client （这里这个 Client 的类型是 `retrofit.client.Client`）。一般来说，也可以不调用 `setClient()`， `RestAdapter` 的 `ensureSaneDefaults()` 方法保证了有默认值可以使用，具体可查看源码。查看源码可以发现 `retrofit.client.Client` 的 `Response execute(Request request)` 方法就是执行 Request 并得到 Response 的地方。那么我们就可以在这里入手做些文章。

#### 实现
用装饰者模式，实现 Client 接口，把本来的 Client 包装一下，来给 `execute()` 方法增加 mock 的职能。

```java
class MyClient implements Client {

    private Client mClient;
    private MockServer mMockServer = null;

    /**
     * @param endPoint 服务器HOST
     * @param realClient 不用mock时，真正的client
     * @param interfaceClass 声明API方法的interface
     */
	public MyClient(String endPoint, Client realClient, Class interfaceClass) {
		if (realClient == null) {
			throw new IllegalArgumentException("real client must not be null!");
		}

		this.mClient = realClient;
        if (Constants.MOCK_RESP_ENABLED) {
            this.mMockServer = new MockServer(endPoint, interfaceClass);
        }
	}

    @Override
    public Response execute(Request request) throws IOException {
        String url = request.getUrl();
        // 由于执行到execute()方法时，如果是GET，request的url已经带了参数，
        // 所以我们需要处理url，取出一个API本身的path
        String path = null;
        try {
            int indexOfQuery = url.indexOf('?');
            if (indexOfQuery > 0) {
                path = url.substring(0, indexOfQuery);
            } else {
                path = url;
            }
        } catch (Exception ignored) {
        }

        Response mockResponse = null;
        // 配置一个是否启用mock的开关，最好配置在build.gradle里，在上线前关闭
        if (Constants.MOCK_RESP_ENABLED) {
			// 根据path来区分是哪个API
            mockResponse = mMockServer.makeMockResponse(url, path);
        }

		// 如果这个API配置了mock，则使用mock，否则请求真实服务器接口
        if (mockResponse != null) {
            return mockResponse;
        } else {
            return mClient.execute(request);
        }
    }
}
```

接下来，`MockServer ` 类的实现思路就很简单了， `makeMockResponse()` 方法针对不同的API（通过 `path` 参数判断）返回相应的假数据即可。用一个 `HashMap<String, String>` 来存储path 对应 mock response 的键值对表，对相应的API根据key查询。

但是为了与 Retrofit 本身使用注解的风格相统一，我们也可以使用注解来标注一个接口方法的Mock。如：

```java
interface MyApiService {

	// ...

	@MyMock(MockResponses.USER_INFO)
	@GET("/api/userInfo")
	void getUserInfo(Callback<String> cb);

	// ...
}
```

```java
interface MockResponses {

	// ...
	
	String USER_INFO = "{\"status\":0,\"message\":\"\",\"data\":{\"nickname\":\"Donald Trump\",\"age\":70}}";
	
	// ...
}
```

自定义注解 `MyMock` ：

```java
package me.tangni.xxx;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

/**
 * @author tangni
 */

@Documented
@Target(METHOD)
@Retention(RUNTIME)
public @interface MyMock {
    String value();
}
```

在 `MockServer` 中，构造方法需要传入声明接口方法的 interface 的类型，本例中就是 `MyApiService.class`，然后通过反射，遍历其中的所有声明的方法，并取出每个方法的 `@GET` 或 `@POST` 注解 （暂时只处理这两个）中的API的path，用作存储 Mock response 的 key。

```java
public class MockServer {

    private HashMap<String, String> mockResps;

    public MockServer(String endPoint, Class interfaceClass) {
        if (Constants.MOCK_RESP_ENABLED) {
            mockResps = generateMockRespMap(endPoint, interfaceClass);
        }
    }

    /**
     *
     * @param reqUrl 请求的url
     * @param path 接口的path
     * @return 如果该接口指定了mock response，则返回mock response，否则返回null。
     */
    public Response makeMockResponse(String reqUrl, String path) {
        if (mockResps != null && mockResps.containsKey(path)) {
            String mockResp = mockResps.get(path);
            return generateMockResponse(reqUrl, mockResp, System.currentTimeMillis());
        } else {
            return null;
        }
    }

    private static HashMap<String, String> generateMockRespMap(String endPoint, Class interfaceClass) {
        HashMap<String, String> map = new HashMap<>();
        Method[] methods = interfaceClass.getDeclaredMethods();
        if (methods != null && methods.length > 0) {
            for (Method method : methods) {
                MyMock mockRespAnno = method.getAnnotation(MyMock.class);
                if (mockRespAnno != null) {
                    String path = null;
                    // TODO: 这里先只处理 GET 和 POST
                    Annotation methodAnno = method.getAnnotation(GET.class);
                    if (methodAnno == null) {
                        methodAnno = method.getAnnotation(POST.class);
                        if (methodAnno != null) {
                            path = ((POST)methodAnno).value();
                        }
                    } else {
                        path = ((GET)methodAnno).value();
                    }

                    if (!TextUtils.isEmpty(path)) {
                        String mockResp = mockRespAnno.value();
                        map.put(endPoint + path, mockResp);
                    }
                }
            }
        }
        return map;
    }

    private static Response generateMockResponse(String url, final String mockResp, long sentMillis) {
        final int contentLen = mockResp != null ? mockResp.length() : 0;

        List<Header> headers = new ArrayList<>();
        headers.add(new Header("Server", "MOCK-SERVER"));
        headers.add(new Header("Content-Type", "application/json;charset=UTF-8"));
        headers.add(new Header("OkHttp-Selected-Protocol", "http/1.1"));
        headers.add(new Header("OkHttp-Sent-Millis", String.valueOf(sentMillis)));
        headers.add(new Header("OkHttp-Received-Millis", String.valueOf(System.currentTimeMillis())));

        TypedInput body = new TypedInput() {
            @Override
            public String mimeType() {
                return "application/json;charset=UTF-8";
            }

            @Override
            public long length() {
                return contentLen;
            }

            @Override
            public InputStream in() throws IOException {
                String content = mockResp;
                if (content == null) {
                    content = "";
                }
                return new ByteArrayInputStream(content.getBytes("UTF-8"));
            }
        };

        return new Response(url, 200, "MOCK-OK", headers, body);
    }
}
```

最后，使用：
```java
RestAdapter restAdapter = new RestAdapter.Builder()
		.setLogLevel(Constants.DEBUG ? RestAdapter.LogLevel.FULL : RestAdapter.LogLevel.NONE)
		.setRequestInterceptor(/*...*/)
		.setConverter(new MyCustomJsonConverter())
		.setClient(new MyClient(Constants.SERVER_HOST, new MyRealClient(), MyApiService.class))
		.setEndpoint(Constants.SERVER_HOST)
		.build();

mApiService = restAdapter.create(MyApiService.class);
```