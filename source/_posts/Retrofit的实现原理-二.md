title: Retrofit的实现原理(二)
date: 2016-01-28 15:22:13
tags: [Retrofit, Network, Android]
---

上篇文章我们已经了解了retrofit的RestAdapter adapter=new RestAdapter.Builder().setEndpoint(url).build()这段代码做了什么。

现在有下面一个接口,
``` Java
interface SimplePOST{
    @POST("/android") 
    Response getResponse();
}
```
这篇我们就来了解下 SimplePOST simplePost= adapter.create(SimplePOST.class)的内部逻辑。

<!-- more -->

``` Java
public <T> T create(Class<T> service) {
    Utils.validateServiceClass(service);
    return (T) Proxy.newProxyInstance(service.getClassLoader(),
        new Class<?>[] { service },      //动态代理
    new RestHandler(getMethodInfoCache(service)));
}

static <T> void validateServiceClass(Class<T> service) {      
//确保create()参数是一个接口类型,且这个接口没有继承其他的接口.(使用动态代理的前提)
    if (!service.isInterface()) {
        throw new IllegalArgumentException(
            "Only interface endpoint definitions are supported.");
    }
    if (service.getInterfaces().length > 0) {
        throw new IllegalArgumentException(
            "Interface definitions must not extend other interfaces.");
    }
}
```
可以发现adapter.create()方法的内部实现是利用动态代理生成了service接口的一个实现类。

根据动态代理的原理. 可以得知调用实现类的方法其实就是调用InvocationtHandler的对应方法。

这里简单介绍下动态代理。(关于动态代理的原理，可以参考其他博客的文章。)

委托类：被代理类即称为委托类。一般不直接调用委托类的方法,而是通过代理类来间接调用。

假定有委托类A，通过动态代理生成了代理类B，生成的时候需要建立一个InvocationHandler H。 实际我们调用代理类B的方法时，方法内部调用的是H的invoke()方法。 而一般H的invoke方法会调用A的同名方法，这样就实现了代理类B功能。

虽然这里是运用了动态代理的技术。但是却和一般的动态代理不一样。 一般的动态代理的InvocationHandler应该通过构造函数中传入委托类A。然后在invoke方法中调用A的方法，但这里是没有委托类的。只是利用动态代理自动生成接口的实现类。

因为java的动态代理是基于接口的，所以retrofit也要求用户自定义的也必须是一个接口。 

注意invocationHandler的invoke()方法执行是在我们调用接口的方法的时候执行的。对于上面的代码就simplepost.getResponse()执行的时候。

``` Java
private class RestHandler implements InvocationHandler {
    private final Map<Method, RestMethodInfo> methodDetailsCache;
 
    RestHandler(Map<Method, RestMethodInfo> methodDetailsCache) {
        //一般的动态代理。InvocationHandler构造函数一般要传入委托类。
        this.methodDetailsCache = methodDetailsCache;
   }
 
    @SuppressWarnings("unchecked") 
    @Override public Object invoke(Object proxy, Method method, final Object[] args)  
    //动态代理的方法实现。调用委托类的方法，最终会调用invoke方法。
    throws Throwable {
        // If the method is a method from Object then defer to normal invocation.
        if (method.getDeclaringClass() == Object.class) {
            //因为接口默认也继承Object。所以接口也有Object中的方法的。 如继承自Object如equals()方法
            return method.invoke(this, args);  
            //这里需要注意下，这里调用的是invocationHandler(H)的方法.一般的动态代理这里应该是调用委托类(A)的方法。
        }
 
    // Load or create the details cache for the current method.
    final RestMethodInfo methodInfo = getMethodInfo(methodDetailsCache, method);  
    //把method和RestMethodInfo缓存起来。防止重复解析method。(把方法的注解，返回值，同步异步等信息都解析出来储存在RestMethodInfo中。)
 
    if (methodInfo.isSynchronous) {  //同步的方式。
        try {
        return invokeRequest(requestInterceptor, methodInfo, args);  
        //核心方法(网络访问)。可以看到retrofit的synchronous类型方法并不是子线程执行的。 所以在Android平台使用同步方式的retrofit的话要在子线程中。
        } catch (RetrofitError error) {
            Throwable newError = errorHandler.handleError(error);
            if (newError == null) {
            throw new IllegalStateException(
                "Error handler returned null for wrapped exception.",  
                //自定义的ErrorHandler的hanleError()方法不能return null。
               error);
            }
            throw newError;  
            //注意,同步的方式，当遇到错误的时候，会抛出一个RuntimeException。在Android下尽量不要使用同步的方式(因为RuntimeException是不提示用户主动捕捉的)。
        }
    }
 
    if (httpExecutor == null || callbackExecutor == null) {
        throw new IllegalStateException(
            "Asynchronous invocation requires calling setExecutors.");
    }
 
    if (methodInfo.isObservable) { //rx的方式
        if (rxSupport == null) {
            if (Platform.HAS_RX_JAVA) {
                rxSupport = new RxSupport(httpExecutor, errorHandler, requestInterceptor);
            } else {
                throw new IllegalStateException(
                    "Observable method found but no RxJava on classpath.");
            }
        }
        return rxSupport.createRequestObservable(new RxSupport.Invoker() {
            @Override public ResponseWrapper invoke(RequestInterceptor requestInterceptor) {
                return (ResponseWrapper) invokeRequest(requestInterceptor, methodInfo, args);
            }
        });
    }
    //同步, Observable方式都不是的话,肯定是异步的方式了。
    // Apply the interceptor synchronously, recording the interception so we can replay it later.
    // This way we still defer argument serialization to the background thread.
    final RequestInterceptorTape interceptorTape = new RequestInterceptorTape();
    requestInterceptor.intercept(interceptorTape);
        
    Callback<?> callback = (Callback<?>) args[args.length - 1];
    httpExecutor.execute(new CallbackRunnable(callback, callbackExecutor, errorHandler) {  
        //可以看到在Android上异步方式是通过HttpExecutor执行的。是默认子线程执行的。 所以异步方式的retrofit不需要在子线程中执行。
        @Override public ResponseWrapper obtainResponse() {
            return (ResponseWrapper) invokeRequest(interceptorTape, methodInfo, args);
        }
    });
    return null; // Asynchronous methods should have return type of void.
}
 
/**
* Execute an HTTP request.
*
* @return HTTP response object of specified {@code type} or {@code null}.
* @throws RetrofitError if any error occurs during the HTTP request.
*/
private Object invokeRequest(RequestInterceptor requestInterceptor,
    RestMethodInfo methodInfo, Object[] args) {
    
    String url = null;
    try {
        methodInfo.init(); // Ensure all relevant method information has been loaded.  --->保证method的参数，返回值，注解都解析完成。 可以得到网络访问的url，header，query，访问方式等信息。
 
        String serverUrl = server.getUrl();
        RequestBuilder requestBuilder = new RequestBuilder(serverUrl, methodInfo, converter);
        requestBuilder.setArguments(args);
 
        requestInterceptor.intercept(requestBuilder);  //在网络访问之前，执行拦截器的方法。 保证拦截器是最后起作用的。
 
        Request request = requestBuilder.build();
        url = request.getUrl();
 
        if (!methodInfo.isSynchronous) {
            // If we are executing asynchronously then update the current thread with a useful name.
        int substrEnd = url.indexOf("?", serverUrl.length());
        if (substrEnd == -1) {
            substrEnd = url.length();
        }
        Thread.currentThread().setName(THREAD_PREFIX
            + url.substring(serverUrl.length(), substrEnd));
        }
 
        if (logLevel.log()) {
            // Log the request data.
            request = logAndReplaceRequest("HTTP", request, args);
        }
 
        Object profilerObject = null;
        if (profiler != null) {
            profilerObject = profiler.beforeCall();
        }
 
        long start = System.nanoTime();
        Response response = clientProvider.get().execute(request);  
        //调用okHttp或者其他Client进行网络访问。并把返回的数据封装进retrofit的Response中。
        long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
 
        int statusCode = response.getStatus(); //返回码
        if (profiler != null) {
            RequestInformation requestInfo = getRequestInfo(serverUrl, methodInfo, request);
            //noinspection unchecked
            profiler.afterCall(requestInfo, elapsedTime, statusCode, profilerObject);
        }
 
        if (logLevel.log()) {
            // Log the response data.
            response = logAndReplaceResponse(url, response, elapsedTime);
        }
 
        Type type = methodInfo.responseObjectType;
 
        if (statusCode >= 200 && statusCode < 300) { // 2XX == successful request   
            //返回码在200到300之间认为是一次成功的网络访问。
            // Caller requested the raw Response object directly.
            if (type.equals(Response.class)) {                           
                //如果方法的返回值类型(封装在responseObjectType)是retrofit的Response的话。 那就不用converter转换。
                if (!methodInfo.isStreaming) {
                // Read the entire stream and replace with one backed by a byte[].
                response = Utils.readBodyToBytesIfNecessary(response);
                }
 
                if (methodInfo.isSynchronous) {  //同步方式
                    return response;         
                }
                return new ResponseWrapper(response, response); //异步的时候返回的是这个。
            }
 
            TypedInput body = response.getBody();
            if (body == null) {
                if (methodInfo.isSynchronous) {
                    //① 结合下面的② ，
                    //返回值类型不同是因为invokeRequest()分别由2个不同的方法调用。
                    //①是由 invoke()调用的。
                    return null;
                }
                return new ResponseWrapper(response, null);
                //② 的情况是异步的。
                //是由callBack模式中的obtainResponse()中调用的。 
                //下面和上面的返回值类型不同的情况皆是如此
            }
 
            ExceptionCatchingTypedInput wrapped = new ExceptionCatchingTypedInput(body);
            try {
                Object convert = converter.fromBody(wrapped, type); 
                //方法的返回值是非Response。那么由Converter把Response转换成对应的类型。
                logResponseBody(body, convert);
                if (methodInfo.isSynchronous) {
                    return convert;
                }
                return new ResponseWrapper(response, convert);
            } catch (ConversionException e) {
                // If the underlying input stream threw an exception, propagate that rather than
                // indicating that it was a conversion exception.
                if (wrapped.threwException()) {
                    throw wrapped.getThrownException();
                }
 
                // The response body was partially read by the converter. Replace it with null.
                response = Utils.replaceResponseBody(response, null);
 
                throw RetrofitError.conversionError(url, response, converter, type, e);
            }
        }
 
        response = Utils.readBodyToBytesIfNecessary(response);
        throw RetrofitError.httpError(url, response, converter, type);
    } catch (RetrofitError e) {
        throw e; // Pass through our own errors.
    } catch (IOException e) {
        if (logLevel.log()) {
            logException(e, url);
        }
        throw RetrofitError.networkError(url, e);
    } catch (Throwable t) {
        if (logLevel.log()) {
            logException(t, url);
        }
        throw RetrofitError.unexpectedError(url, t);
    } finally {
        if (!methodInfo.isSynchronous) {
            Thread.currentThread().setName(IDLE_THREAD_NAME);
        }
    }
}
```

对于上面的getMethodInfo()方法, 是把method信息提取出来,封装到RestMethodInfo中。 包括访问方式。Headers，path，参数，返回值等。
``` Java
static RestMethodInfo getMethodInfo(Map<Method, RestMethodInfo> cache, Method method) {
    synchronized (cache) {
        RestMethodInfo methodInfo = cache.get(method);
        if (methodInfo == null) {                  
            //经典的逻辑，如果缓存中有就从缓存中拿，如果没有,就生成一个新的，并且放到缓存中。
            methodInfo = new RestMethodInfo(method);
            cache.put(method, methodInfo);
        }
        return methodInfo;
    }
}
```

``` Xml
<strong>RestMethodInfo</strong>
```

``` Java
final class RestMethodInfo {
    final Method method;
 
    boolean loaded = false;   //方法是否已经load过(解析过)
 
    // Method-level details
    final ResponseType responseType;
    final boolean isSynchronous;      //方法是同步还是异步
    final boolean isObservable;
    Type responseObjectType;
    RequestType requestType = RequestType.SIMPLE;
    String requestMethod;
    boolean requestHasBody;
    String requestUrl;           
    Set<String> requestUrlParamNames;
    String requestQuery;
    List<retrofit.client.Header> headers;
    String contentTypeHeader;
    boolean isStreaming;
    private enum ResponseType {  //方法的返回值是什么类型
        VOID,                    //void代表没有返回值，-->异步的方式
        OBSERVABLE,              //rxjava
        OBJECT                   //方法的返回值是对象--->同步的方式
    }
    RestMethodInfo(Method method) {
        this.method = method;
        responseType = parseResponseType();
        isSynchronous = (responseType == ResponseType.OBJECT);
        isObservable = (responseType == ResponseType.OBSERVABLE);
    }
 
    private ResponseType parseResponseType() {
        // Synchronous methods have a non-void return type.          
        //同步的方法有一个非void的返回值
        // Observable methods have a return type of Observable.
        //Observable的方法返回值类型应该是Observable
        Type returnType = method.getGenericReturnType();
 
        // Asynchronous methods should have a Callback type as the last argument.  
        //异步的方法最后一个方法的参数类型应该是Callback。
        Type lastArgType = nul;
        Class<?> lastArgClass = null;
        Type[] parameterTypes = method.getGenericParameterTypes();
        if (parameterTypes.length > 0) {
            Type typeToCheck = parameterTypes[parameterTypes.length - 1];
            lastArgType = typeToCheck;
            if (typeToCheck instanceof ParameterizedType) {
                typeToCheck = ((ParameterizedType) typeToCheck).getRawType();
            }
            if (typeToCheck instanceof Class) {
                lastArgClass = (Class<?>) typeToCheck;
            }
        }
 
        boolean hasReturnType = returnType != void.class;
        boolean hasCallback = lastArgClass != null && 
            Callback.class.isAssignableFrom(lastArgClass); //如果有CallBack， CallBack只能是最后一个参数。
 
        // Check for invalid configurations.
        if (hasReturnType && hasCallback) {                         
            //返回值是非void类型和方法有CallBack参数有且只能有一个满足
            throw methodError(
                "Must have return type or Callback as last argument, not both.");
        }
        if (!hasReturnType && !hasCallback) {
            throw methodError(
                "Must have either a return type or Callback as last argument.");
        }
 
        if (hasReturnType) {
            if (Platform.HAS_RX_JAVA) {
                Class rawReturnType = Types.getRawType(returnType);
                if (RxSupport.isObservable(rawReturnType)) {
                returnType = RxSupport.getObservableType(returnType, rawReturnType);
                responseObjectType = getParameterUpperBound((ParameterizedType) returnType);
                return ResponseType.OBSERVABLE;
                }
            }
            responseObjectType = returnType;
            return ResponseType.OBJECT;
        }
 
        lastArgType = Types.getSupertype(lastArgType,
            Types.getRawType(lastArgType), Callback.class);
        if (lastArgType instanceof ParameterizedType) {
            responseObjectType = getParameterUpperBound((ParameterizedType) lastArgType);
            return ResponseType.VOID;
        }
 
        throw methodError(
            "Last parameter must be of type Callback<X> or Callback<? super X>.");
    }

    private void parsePath(String path) {
        //解析路径 如GET(string)，注意这里会对string进行判断,必须以"/"开头。所以不能是空字符串，如""
        ...........
    }
    List<retrofit.client.Header> parseHeaders(String[] headers) { //解析Headers注解。
　　     ...........
    }
    private void parseParameters() {   //解析方法参数，如方法参数的@Header，@Query等。
　　     ...........
    }
    private void parseMethodAnnotations() { //解析http方式。如GET()，POST()
　　     ...........
    }
}
```