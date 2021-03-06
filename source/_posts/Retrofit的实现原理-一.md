title: Retrofit的实现原理(一)
date: 2015-11-17 07:26:39
tags: [Retrofit, Network, Android]
---

Retrofit框架实现的这么巧妙,虽然我们不需要再造一个轮子,但研究下轮子的实现还是很有帮助的。

Retrofit有几个关键的地方：

- 用户自定义的接口和接口方法.(由动态代理创建对象.)
- Converter转换器.(把response转换为一个具体的对象)
- 注解的使用.

先从API来看。

<!-- more -->

```java
RestAdapter restAdapter = new RestAdapter.Builder().setEndpoint(API_URL).build(); 
```

build()其内部实现是这样的:

```java
public RestAdapter build() {
    if (endpoint == null) {
        throw new IllegalArgumentException("Endpoint may not be null.");
    }
    ensureSaneDefaults();
    return new RestAdapter(endpoint, clientProvider, httpExecutor, 
        callbackExecutor, requestInterceptor, converter, profiler, 
        errorHandler, log, logLevel);
}
```

当用户没有设置自定义的converter，client， httpExecutor（http访问执行的线程只对异步的retrofit有效）， callBackExecutor（异步的callBack执行的线程），errorHandler，log， RequestInterceptor的时候，就会使用retrofit默认的配置。

```java
private void ensureSaneDefaults() {
    if (converter == null) {
        converter = Platform.get().defaultConverter();
    }
    if (clientProvider == null) {
        clientProvider = Platform.get().defaultClient();
    }
    if (httpExecutor == null) {
        httpExecutor = Platform.get().defaultHttpExecutor();
    }
    if (callbackExecutor == null) {
        callbackExecutor = Platform.get().defaultCallbackExecutor();
    }
    if (errorHandler == null) {
        errorHandler = ErrorHandler.DEFAULT;
    }
    if (log == null) {
        log = Platform.get().defaultLog();
    }
    if (requestInterceptor == null) {
        requestInterceptor = RequestInterceptor.NONE;
    }
    }
}
```

Platform.java

```java
private static final Platform PLATFORM = findPlatform();

static Platform get() {
    return PLATFORM;
}

private static Platform findPlatform() {
    try {
        Class.forName("android.os.Build");//只要android.os.Build的class可以正常找到,证明是在android平台
        if (Build.VERSION.SDK_INT != 0) {
            return new Android();
        }
    } catch (ClassNotFoundException ignored) {
    }

    if (System.getProperty("com.google.appengine.runtime.version") != null) {
        return new AppEngine();   //google的app Engine平台
    }

    return new Base();
}　　
```

Retrofit的Android类继承了Platform，根据android的特性对配置项做了处理。

```java
private static class Android extends Platform {
    @Override Converter defaultConverter() {
        return new GsonConverter(new Gson());  //默认的转换器是Gson
    }

    @Override Client.Provider defaultClient() {
        final Client client;
        if (hasOkHttpOnClasspath()) {  //有okhttp的路径就使用 Okhttp
            client = OkClientInstantiator.instantiate();
        } else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.GINGERBREAD) {
            client = new AndroidApacheClient(); //没有okhttp,且版本小于2.3 使用HttpClient
        } else {
            client = new UrlConnectionClient();  //没有okhttp,且版本大于等于2.3 使用urlConnection.
        }
        return new Client.Provider() {
            @Override public Client get() {
                return client;
            }
        };
    }

    @Override Executor defaultHttpExecutor() {   //网络访问执行的线程.
        return Executors.newCachedThreadPool(new ThreadFactory() {  //一个cached的线程池.可以复用老线程且线程长时间不用会自动回收. 线程池中线程不够会生成新线程.
            @Override public Thread newThread(final Runnable r) {
                return new Thread(new Runnable() {
                    @Override public void run() {
                        Process.setThreadPriority(THREAD_PRIORITY_BACKGROUND);    //设置线程的优先级 为最低
                        r.run();
                    }
                }, RestAdapter.IDLE_THREAD_NAME);
            }
        });
    }

    @Override Executor defaultCallbackExecutor() { //异步执行的线程.
        return new MainThreadExecutor();
    }

    @Override RestAdapter.Log defaultLog() {  //通过Log.d("Retrofit",String)打印log
        return new AndroidLog("Retrofit");
    }
}
```

```java
private static boolean hasOkHttpOnClasspath() {
    try {
        Class.forName("com.squareup.okhttp.OkHttpClient"); //是否可以找到OkHttpClient类.
        return true;
    } catch (ClassNotFoundException ignored) {
    }
    return false;
}
```

可以发现上面默认的Http的Executor是一个线程池。
而CallBack的Executor是在主线程执行的。由绑定MainLooper的Handler提交到主线程执行。

```java
public final class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper()); //关联主线程的Handler

    @Override public void execute(Runnable r) {
        handler.post(r);                                 //提交到主线程执行
    }
}
```

```java
public final boolean post(Runnable r) {
    return  sendMessageDelayed(getPostMessage(r), 0);
}
```

```java
private static Message getPostMessage(Runnable r) {   //把runnable封装到Message中.
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
