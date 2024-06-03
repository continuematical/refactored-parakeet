简介

支持RESTful API设计风格，网络请求实际工作由OkHttp完成，Retrofit主要负责请求接口的***包装***；

![img](https://ask.qcloudimg.com/http-save/yehe-2802329/w6go1edupm.png?imageView2/2/w/2560/h/7000)

# RESTful API

即表述性状态转移，只是风格没有标准，其核心是一切资源（数据加表现形式的集合），任何可命名的抽象概念都可以定义为一个资源，符合REST风格的API即为RESTful API；

# RESETful 对资源的操作

***POST***

在服务器新建一个资源，对应资源操作是INSERT，非幂等且不安全；

***GET***

从服务器中取出资源，对应资源操作是SELECT，幂等且安全；

***DELETE***

从服务器删除资源，对应资源操作是DELETE，幂等且不安全；

***PUT***

在服务器更新资源，客户端提供改变后的完整资源，对应资源操作是UPDATE，幂等且不安全；

*安全*是否会修改服务器的状态；

*幂等*返回数据是否相等；

# Retrofit的注解

## HTTP请求方法注解

*@GET*

获取数据

*@POST*

添加数据项

*@PUT*

实现数据项的更新

*@HTTP*

作用：替换*@GET、@POST、@PUT、@DELETE、@HEAD*注解的作用及更多功能拓展

自定义请求方式，是否有请求体等

```java
@HTTP(method="GET"，hasBody=false)
```

```java
public @interface HTTP {
  String method();
  String path() default "";
  boolean hasBody() default false;
}
```

*@DELETE*

DELETE注解一般必须添加相对路径或绝对路径或者全路径，如果不想在DELETE注解后添加请求路径，则可以在方法的第一个参数中用@Url注解添加请求路径；

## 标记类注解

*FormUrlEncoded*

修饰`@Field` 和`@FieldMap` 参数；

表明这是一个表单请求，发送*form-encoded*数据，*@Field*里的数据才会被接入参数中，自动创建一个`FormBody`对象，每个键-值对都要被含有名字的@Field注解和提供值的对象所标注；

*Multipart*

支持多个*@Part*，将会发送multipart数据，Parts都使用@Part注解进行声明；要使用Retrofit的众多转换器之一或者实现RequestBody来处理自己的序列化；

*Streaming*

代表响应的数据以流的形式返回（适用于返回数据较大的场景），如果没有该注解，默认会存储到内存中，之后读取数据也是从内存中载入；

## 参数类注解

*@Query*

动态指定查询条件

*@QueryMap*

动态指定查询条件组

*@Field*

一定要加上*@FormUrlEncode* ，不然会报错；

*POST*请求中传入键值对数值段，用于发送一个表单请求；

用`String.valueOf()` 把参数值转换为String，然后进行URL编码,当参数值为null值时，会自动忽略，如果传入的是一个*List*或*array*，则为每一个非空的*item*拼接一个键值对，每一个键值对中的键是相同的，值就是非空item的值，如: `name=张三&name=李四&name=王五` ，另外,如果item的值有空格，在拼接时会自动忽略，例如某个item的值为:`张 三` ，则拼接后为`name=张三` 

*@FieldMap* 

发送于一个表单请求；

*Map*中每一项键和值都不能为空，否则会报出异常；

```java
//普通参数 
@FormUrlEncoded
@POST("/")
Call<ResponseBody> example(@Field("name") String name,@Field("occupation") String occupation);
 //固定或可变数组
@FormUrlEncoded 
@POST("/list") 
Call<ResponseBody> example(@Field("name") String... names);
```

*@Body*

使用这个注解，将Java对象（自定义数据类型）转化为*JSON*字段发送到服务器；

 *@Body* 和*@Field* 不能一起使用，并且不能与*@FormUrlEncode* （表单请求，会发生冲突）或@MultiPart 结合使用；

当你发送一个*POST/PUT*请求但是不想用表单传输，可以传递一个实体类，`retrofit` 会将实体类序列化并将序列化的结果直接发送过去；

如果提交的是一个*Map*，相当于*@Field*，但是Map必须封装成`FormBody` 

```java
FormBody formBody=new FormBody.Builder().build();
```

**JSON请求和Form表单请求的区别**

1. `content_type` 不一致；
2. 数据格式不一致；

retrofit传递参数一般如下：

```java
 @FormUrlEncoded
 @POST("xxxxxxx")
 Call<Object> login( @Field("参数1") String reason，@Field("参数2") String reason);
```

底层会自动封装一个请求体，并通过注解将参数字符串传递给后台；

如果提交JSON数据如下：

```java
@POST("xxxxxxx")   
Call<Object> login( @Body JSONObject parmas );  
```

无表单参数，相当于Java中的bean对象；

但如果参数过多，按照第一种方式写代码量很大，我们可以自己封装一个请求体`RequestBody` ，可以如下封装：

```java
//创建HashMap并插入数据
public HashMap<String, String> login(String xxx...) {
HashMap<String, String> hashMap = new HashMap<>();
  hashMap.put("参数1", xxx);
  hashMap.put("参数2", xxx);
  hashMap.put("参数3", xxx);
  hashMap.put("参数4", xxx);
  hashMap.put("参数5", xxx);
  hashMap.put("参数6", xxx);
  return hashMap;
}
```

```java
public RequestBody getRequestBody(HashMap<String, String> hashMap) {
StringBuffer data = new StringBuffer();
if (hashMap != null && hashMap.size() > 0) {
  Iterator iter = hashMap.entrySet().iterator();
  while (iter.hasNext()) {
    Map.Entry entry = (Map.Entry) iter.next();
    Object key = entry.getKey();
    Object val = entry.getValue();
      //需要转化为键值对形式
    data.append(key).append("=").append(val).append("&");
  }
}
    String jso = data.substring(0, data.length() - 1);
    //封装
    RequestBody requestBody =
    RequestBody.create(MediaType.parse("application/x-www-form-urlencoded; charset=utf-8"),jso);
    //第一个参数是MediaType，是媒体类型，第二个参数类型可以是String，File等
    return requestBody;
 }
```

**MediaType**决定浏览器将以什么形式，什么编码方式对图片进行解析；

[HTTP Content-type 对照表 (oschina.net)](https://tool.oschina.net/commons)

@Part*

单个文件上传；

*@PartMap*

用Map封装上传文件；

## 其他注解

*@Header*

动态添加消息报头，必须给*@Header*提供相应的参数，如果参数的值为空header将会被忽略，否则就调用参数值的`toString()` 方法并使用返回结果；

当传入一个`List` 或`array` 时，拼接每个非空的`item` 的值到请求头中；

具有相同名称的请求头不会相互覆盖，而是会照样添加到请求头中；

*@Headers*

添加多个静态请求头；

*@Path*

动态配置Uri地址

*@Url*

传入的地址不需要加上*@Post*里的端口；

![img](https://ask.qcloudimg.com/http-save/yehe-2802329/l0ei6barij.jpeg?imageView2/2/w/2560/h/7000)

# 转换器和适配器

## 转换器

接收到服务器的响应后，目前无论是`OkHttp`和`Retrofit`都只能接收到***String***类型的数据，实际开发中，我们需要将字符串转化为***Java Bean***对象，比如服务器响应数据对象为***JSON***格式字符串，那么我们可以用***GSON***库来进行反序列化的操作，Retrofit提供了多种转化库来完成数据的转换；

***自定义创建***

***添加转换器***

```java
addConverterFactory(GsonConverterFactory.create())
```

## 适配器

### 嵌套请求

实际过程中可能遇到先申请A接口，再申请B接口，请求具有先后顺序；

retrofit网络请求返回对象必须是`Call` ，申请完A接口后，我们需要把Call对象转化为Java对象来进行下一个接口请求。如果我们需要返回的不是Call对象，就需要***适配器***来解决这个问题。

### 转换器

# 日志截断

`HttpLoggingInterceptor` 

`setLevel()` 

| 级别   | 说明                       |
| ------ | -------------------------- |
| Body   | 记录所有的东西             |
| Header | 记录请求和相应行，以及标头 |

# 文件

### 单个文件

可以使用`RequestBody` 封装图片

### 文件和字段

`MultipartBody.Part` 

# 解析

### 添加依赖

### 设置权限

```xml
<!--AndroidManifest.xml-->
<uses-permission android:name="android.permission.INTERNET"/>
```

### 创建接口

1. 用 动态代理 动态 将该接口的注解“翻译”成一个 Http 请求，最后再执行 Http 请求；
2. 注：接口中的每个方法的参数都需要使用注解标注，否则会报错；

```java
public interface GetRequest_Interface {

    @GET("openapi.do?keyfrom=Yanzhikai&key=2032414398&type=data&doctype=json&version=1.1&q=car")
    Call<Translation>  getCall();
    // @GET注解的作用:采用Get方法发送网络请求

    // getCall() = 接收网络请求数据的方法
    // 其中返回类型为Call<*>，*是接收数据的类（即上面定义的Translation类）
    // 如果想直接获得Responsebody中的内容，可以定义网络请求返回值为Call<ResponseBody>
}
```

### Retrofit对象

```java
mRetrofit = new Retrofit.Builder()
                .baseUrl("https://api.github.com/")
                .addConverterFactory(GsonConverterFactory.create())
    //转换器：Java对象和Json形式数据的转换
                .build();

        ApiService service = mRetrofit.create(ApiService.class);
        Call<List<GithubRepo>> call = service.get();
        call.enqueue(new Callback<List<GithubRepo>>() {
            @Override
            public void onResponse(Response<List<GithubRepo>> response, Retrofit retrofit) {
                Log.i(TAG, "get+" + response.body().get(0));
                text.setText(response.body().get(0).toString());
            }

            @Override
            public void onFailure(Throwable t) {
                Log.i(TAG, "failure"+t.toString());
            }
        });
```

## Retrofit的创建过程

配置retrofit里的成员变量；

```java
mRetrofit = new Retrofit.Builder()
                .baseUrl("https://api.github.com/")
                .addConverterFactory(GsonConverterFactory.create())
                .build();
```

### Facade外观设计模式

### Builder构建者设计模式

创建一个对象时，可能不能拥有该对像的全部信息。如创建retrofit对象，需要逐步获得网络请求地址、转换工厂等信息（存在可选参数），更方便的方法是***分步构建对象***；

或者是类的构造器过于复杂（构造函数参数过多），耦合性比较高的时候使用；

```java
//回调方法执行器，处理网络请求
//OkHttp切换线程需要手动调用，但是retrofit不用
public Builder callbackExecutor(Executor callbackExecutor) {
      this.callbackExecutor = checkNotNull(callbackExecutor, "callbackExecutor == null");
      return this;
    }
```

```java
public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }//baseUrl必需

      OkHttpClient client = this.client;
      if (client == null) {
        client = new OkHttpClient();
      }
    
    //adapterFactories网络请求适配器
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
    
    //Platform传回一个转化器,默认添加defaultCallAdapterFactory
    //callbackExecutor用来将回调传递到UI线程
      adapterFactories.add(Platform.get().defaultCallAdapterFactory(callbackExecutor));

    //converterFactories存储对Call进行转化的对象
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(client, baseUrl, converterFactories, adapterFactories, callbackExecutor,validateEagerly);
    }
```

```java
//将GsonConverFactory加入Converter.Factory里，表示返回的数据支持json格式
public Builder addConverterFactory(Converter.Factory converterFactory) {
      converterFactories.add(checkNotNull(converterFactory, "converterFactory == null"));
      return this;
    }
```

```Platform```

```java
private static final Platform PLATFORM = findPlatform();
//获取平台
  static Platform get() {
    return PLATFORM;
  }
//根据不同的平台调用不同的线程池
  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }

CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapter.FACTORY;
  }
```

## Call的创建过程

```java
ApiService service = mRetrofit.create(ApiService.class);
```

### Proxy代理设计模式

```java
//T为接口类型
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    //动态代理：在程序运行时创建
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },//类加载器 Class对象数组 调用处理器
        new InvocationHandler() {
          private final Platform platform = Platform.get();
									//代理对象			调用方法		方法参数
          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //返回接口类型的对象
            return loadMethodHandler(method).invoke(args);
          }
        });
  }
```

```java
//loadMethodHandler(method)
MethodHandler<?> loadMethodHandler(Method method) {
    MethodHandler<?> handler;
    			//查询是否具有缓存
    synchronized (methodHandlerCache) {
      //是否具有缓存，如果没有，就创建一个，并且加入methodHandlerCache
      handler = methodHandlerCache.get(method);
      if (handler == null) {
        handler = MethodHandler.create(this, method);
        methodHandlerCache.put(method, handler);
      }
    }
    return handler;
  }
```

```java
//handler构建
static MethodHandler<?> create(Retrofit retrofit, Method method) {
    //创建callAdapter，Call转换后的对象
    CallAdapter<Object> callAdapter = (CallAdapter<Object>) createCallAdapter(method, retrofit);
    //数据的真实类型,比如传入的是Call<String>，返回的就是String
    Type responseType = callAdapter.responseType();
    //遍历ConvertFactory里的Factory，返回一个合适的convert来转换对象
    Converter<ResponseBody, Object> responseConverter =(Converter<ResponseBody, Object>) createResponseConverter(method, retrofit, responseType);
    RequestFactory requestFactory = RequestFactoryParser.parse(method, responseType, retrofit);
    return new MethodHandler<>(retrofit, requestFactory, callAdapter, responseConverter);
  }
```

```java
//createCallAdapter
private static CallAdapter<?> createCallAdapter(Method method, Retrofit retrofit) {
    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw Utils.methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
      throw Utils.methodError(method, "Service methods cannot return void.");
    }
    //请求方式（GET，PUT），
    Annotation[] annotations = method.getAnnotations();
    try {
      return retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw Utils.methodError(e, method, "Unable to create call adapter for %s", returnType);
    }
  }

//callAdapter
public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }


public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

	//返回adapterFactories（转换器）列表
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
```

```java
//ExecutorCallAdapterFactory  get
public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (Utils.getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```

```java
//createResponseConverter
private static Converter<ResponseBody, ?> createResponseConverter(Method method,
      Retrofit retrofit, Type responseType) {
    Annotation[] annotations = method.getAnnotations();
    try {
      return retrofit.responseConverter(responseType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw Utils.methodError(e, method, "Unable to create converter for %s", responseType);
    }
  }

//retrofit.responseConverter
//遍历converterFactories，并返回一个合适的Converter来转换Call
public <T> Converter<ResponseBody, T> responseConverter(Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");
    checkNotNull(annotations, "annotations == null");

    for (int i = 0, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).fromResponseBody(type, annotations);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
```

```java
//invoke()
Object invoke(Object... args) {
    //新建一个OKHttpCall，进行赋值操作，retrofit的Call实际上是对OkHttpCall的一个封装，封装过程由adapter（适配器）完成；
    return callAdapter.adapt(new OkHttpCall<>(retrofit, requestFactory,responseConverter, args));
  }
```

```java
//OkHttpCall构造方法，赋值
OkHttpCall(Retrofit retrofit, RequestFactory requestFactory,
      Converter<ResponseBody, T> responseConverter, Object[] args) {
    this.retrofit = retrofit;
    this.requestFactory = requestFactory;
    this.responseConverter = responseConverter;
    this.args = args;
  }
```

## Call.enqueue

```java
//OkHttpCall  enqueue
@Override public void enqueue(final Callback<T> callback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed");
      executed = true;
    }

    com.squareup.okhttp.Call rawCall;
    try {
      rawCall = createRawCall();
    } catch (Throwable t) {
      callback.onFailure(t);
      return;
    }
    if (canceled) {
      rawCall.cancel();
    }
    this.rawCall = rawCall;

    //调用 com.squareup.okhttp.Call.enqueue
    rawCall.enqueue(new com.squareup.okhttp.Callback() {
      private void callFailure(Throwable e) {
        try {
          callback.onFailure(e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      //相应成功
      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(response, retrofit);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
        @Override public void onFailure(Request request, IOException e) {
        callFailure(e);
      }

      @Override public void onResponse(com.squareup.okhttp.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }
    });
  }
```

```java
//parseResponse
private Response<T> parseResponse(com.squareup.okhttp.Response rawResponse) throws 		 		IOException {
    ResponseBody rawBody = rawResponse.body();
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();
	//对返回的不同条件码做不同的操作
    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        ResponseBody bufferedBody = Utils.readBodyToBytesIfNecessary(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        closeQuietly(rawBody);
      }
    }
    //成功
    if (code == 204 || code == 205) {
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```









