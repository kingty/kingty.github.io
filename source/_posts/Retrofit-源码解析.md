---
title: Retrofit 源码解析
date: 2018-06-08 10:11:19
tags: android
---


## 简单用法
Retrofit最简单的用法就是定义一个接口，创建`Retrofit`对象，调用`create()`方法得到一个`service`,
然后自己根据`service`中的方法去做同步或者异步的请求，拿到数据对象，十分简单快速，简单代码如下：

<!--more-->
```java 
public interface GitHub {
   @GET("/repos/{owner}/{repo}/contributors")
   Call<List<Integer>> contributors(@Path("owner") String owner,@Path("repo") String repo);
}

.创建
Retrofit retrofit = new Retrofit.Builder().baseUrl("xxx").build();
.代理
GitHub gitHub = retrofit.create(GitHub.class);
Call<List<Integer>> call = gitHub.contributors("xx", "xx");
.执行
call.enqueue(new Callback<List<Integer>>() {
    @Override
    public void onResponse(Call<List<Integer>> call, Response<List<Integer>> response) {

    }
    @Override
    public void onFailure(Call<List<Integer>> call, Throwable t) {

    }
});
```
## 流程分析

### 创建

那这么简单的过程，刚开始看的时候觉得有点懵é¼呀，怎么他就帮你完成了请求，你明明什么都没有做，下面我们按照它的流程慢慢来解析一下整个过程。
我们要用`Retrofit`,首先自然是要创建它,也就是这行代码`Retrofit retrofit = new Retrofit.Builder().baseUrl("xxx").build();`.
这里创建`Retrofit`是通过它的一个内部类`Builder`来创建的，也就是创建者模式，这个模式很简单，不知道的自行百度，谷歌。
好，我们来看看这个`builder`做了什么,除了初始化有个`Platform.get()`,直接看最后的`build()`,其余的方法都是设置参数，主要就是这个`build()`：

```java
public Builder() {
      this(Platform.get());
}
public Retrofit build() {
  1.
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }
  2.
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }
  3.
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }
  4.
      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
 5.
      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    } 

```
首先一初始化，就做了一件事情，是啥`Platform.get()`，`Platform`是啥？直译过来就是平台啊，平台是啥？为啥要有平台？看下面这个代码`get()`其实
就是一个就是调用`findPlatform()`：

```java
private static final Platform PLATFORM = findPlatform();

static Platform get() {return PLATFORM;}

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
  
  ```

  有`Android`和`Java8`两种平台，`Android`我们还是能理解的，为啥还有个`Java8`，不要问我，我也不知道啊，`Retrofit`的作者炒鸡暖男，关心全世界各种码农，
  我是个写android代码的我们就看`Android`平台就好了
  下面我们来看一下`build()`这个方法：
- 第一步很简单，没有`baseUrl`抛出异常，最基本的没有，事情没法干是吧。
- 第二步，如果没有给他设置`callFactory`，那默认给他一个`callFactory`，默认就是新创建一个`OkHttpClient`，这里可能我们会有自己做过一些
处理的`OkHttpClient`,比如加了`Interceptor`啊之类的，设置进来就好了，就不会用默认的。
有人可能会问啥是`callFactory`啊 ？`callFactory`嘛，就是call的factory嘛，call是啥，就是请求，factory是啥，就是工厂，`callFactory`就是创建请求的
工厂，`OkHttpClient`就是一个很牛逼的创建请求的工厂，不在本文讨论范围内，就不多言了。
- 第三步，设置`callbackExecutor`,又来一个，这`callbackExecutor`又是啥呢？`callback`就是回调嘛，啥回调，就是网络请求返回回来数据的回调，`executor`呢，就是执行者
，合起来就是回调的执行者，意思网络成功了之后交给他它了。如果你没有设置它就自己整一个默认的回调嘛，不能没有。但是这里它要搞事情了，它返回了一个啥？
`platform.defaultCallbackExecutor();`来，我们看一下`android`下它返回的是啥：
```java
static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```
是啥？`MainThreadExecutor()`啊，啥意思就是主线程啊，下面写的明明白白的`Looper.getMainLooper()`再把要执行的`Runnable`post到主线程上执行
因为它是默认的嘛，可能就说大多数人都是得到数据更新UI啊啥的，所以就默认在主线程上执行回调了。我就不想拿到数据在主线程座做咋办，我拿到数据我就想更新数据库，
我想在IO线程上搞事情，那就自己写个`callbackExecutor`，自己在IO线程上做就好了，人家提供了一个方法`callbackExecutor(Executor executor)`给你，你自己设置进去就好了

- 第四步是啥？看代码说话，那就是设置`callAdapterFactory`啊 。`callAdapterFactory`又是什么鬼啊，和上面一样啊，拆分一下呀。`CallAdapter`啥意思，就是请求的
适配器，请求的适配器是什么鬼啊。来来来我告诉你，你看看源码里面根目录是不是有一个包名字叫做`'retrofit-adapter'`,这个包就是实现了一些列的`CallAdapter`
意思就是你想将返回的数据用什么东西包装起来，比如你用`Rxjava`的话想返回`Observable`，或者高兴，想用`Java8`的`CompletableFuture`，这些都由你呀。
但是这些都实现了一个叫`CallAdapter`的接口。我们来简单看看这个接口：
```java
public interface CallAdapter<R, T> {
  Type responseType();
  T adapt(Call<R> call);
  abstract class Factory {
    public abstract CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```
其实接口里面就是两个方法还有一个静态的工厂类。`responseType()`这个方法决定请求回来之后返回的是什么类型的数据。比如在示例用法中我们的`List<Integer>`
`adapt()`这个方法是干嘛的呢？就是适配嘛，就是怎样把返回回来的数据通过这个方法包装成你想要的对象。
这里看到这个名字`adapter`你想到了啥，其实就是传说中的适配器模式啊，就是我给你定义一个接口放这里，我在框架里的逻辑就用这个接口来做就好了，至于你想要怎样的实现，
想用框架供给你的一些实现比如`Rxjava`或者`Java8`的`CallAdapter`,或者是你自己心情好想用自己的实现一个其他的`CallAdapter`，你自己决定就好了。这就是传说中的啥？？扩展性好啊。
继续看`build()`
这个方法，它调用的是`adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));·
再回来`Platform`看`defaultCallAdapterFactory()`返回的是一个`ExecutorCallAdapterFactory`。这个类他么的又来干嘛，当然是搞事情。
进去瞅一眼，发现了什么？它当然是继承`CallAdapter.Factory`了，这个不说了，看几句代码来，看它的`get()`方法，看看这个工厂是怎么造`CallAdapter`的：
```java
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
  ```
  返回了一个简单新创建的实现`CallAdapter`ç匿名类。注意看看这里的`adapt()`方法，前面讲了就是用它来实现到底返回什么包装对象的逻辑。这里返回的是一个
  `ExecutorCallbackCall`,`ExecutorCallbackCall`是这`ExecutorCallAdapterFactory`里面的一个内部类.来看看它的代码：

  ```java
  static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }

    @Override public boolean isExecuted() {
      return delegate.isExecuted();
    }

    @Override public Response<T> execute() throws IOException {
      return delegate.execute();
    }

    @Override public void cancel() {
      delegate.cancel();
    }

    @Override public boolean isCanceled() {
      return delegate.isCanceled();
    }

    @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
    @Override public Call<T> clone() {
      return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
    }

    @Override public Request request() {
      return delegate.request();
    }
  }
  ```
  它实现了`Call`这个接口，`Call`我们前面说了，是啥，就是一个请求嘛，然而我们看这里并没有实际做请求而是用了一个静态代理，
  通过代理类的实现来实现call请求，而在这里面做了一些其他的逻辑比如`cancel`的逻辑，而实际上做请求的还是交个了`delegate -> OkHttpCall`.

- 第五步，接着看上面的`build()`的代码，不要着急，第一段代码还没讲完呢。第五步是什么？`List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);`
这一步就是关于`Converter`，顾名思义，它就是一个转换器，什么转换器，数据的转换器，我们从后端获取到的数据，一般都是一些序列化的数据，例如`json`,`xml`,`protobuf`之类的
而æ们前端用到的需要的是一个对象，我们就需要吧这些序列化的数据转换成我们想要的能直接用的用起来爽的对象，这时候就需要现在登场的这个东西。现在`json`用的
比较多，我们平时都会用什么`gson`,`jackson`或者其他的三方库来转化它，你觉得哪个用起来高兴就可以用什么写一个`Converter`,然后用`Builder`中的`addConverterFactory`
就可以用你想要的了，而且你都不用写，因为官方提供了好多种`Converter`的，在根目录下的`'retrofit-converters'`这个包下面，你只需要用就好了，那我们这里如果没有设置过`converterFactories`
咋办？咋办？没设置，后面找不到会**报错的**。
这里的`Response`是`Retrofit`对`OkHttp`的`ResponseBody`封装了一些逻辑的类，源码就不贴了，自己点进去看看。
这里我们顺便看看`Converter`这个接口：
```java
public interface Converter<F, T> {
  T convert(F value) throws IOException;
  abstract class Factory {
    
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }
    public Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }
    public Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }
  }
}
```
接口就一个方法，就是转换，然后里面还有一个静态的工厂类我们看到里面有3个方法，其实很好理解。我们需要把返回来的`ResponseBody`里的数据转换
成我们想要的东西，我们也会想要把我们`RequestBody`程序里的东西转换成后端想要的东西.就是这个逻辑拉，这个工厂类就是给我们提供各种转换器，我们
只需要根据我们自己的需求来实现或者使用对应的就好了。这又是啥，还是和上面一样啊ï¼给你定义一个接口，接口是什么，就是标准，给你一个标准
你实现这个标准就行，我用我这套标准来实现我内部的逻辑，至于你怎么实现，想用啥方法实现，玩成什么花样都可以，我不管，只要你遵循了标准，就可以。这样
扩展性就好呀。这就是人家大神牛逼之处啊，代码写到高处就是写标准啊。
讲到这里，我们示例用法中的第一句`Retrofit retrofit = new Retrofit.Builder().baseUrl("xxx").build();`总算讲完了。中间这么多逻辑，这么
多心血，你看，你一句话就搞定了，是不是该学习学习。

### 代理

build好了之后，就是需要的材料都搞齐了，要工厂有工厂要材料有材料，下面我们来讲讲这第二句，第二句，那厉害了。其实他就是啥，利用你定义的一个充满各种注解的接口`interface GitHub()`来简单粗暴的做了一个动作，
那就是`create()`。这个动作看似简单，实则过于粗暴啊，进去ç看代码

```java
public <T> T create(final Class<T> service) {
  1.
    Utils.validateServiceInterface(service);
  2.
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            3.
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            4.
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            5.
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            6.
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
这里你首先要了解的知识是泛型，反射，动态代理。如果不懂，请自行google.好吧，我说下动态代理，动态代理就是动态的代理，就是只要你实现了一个借口，`Proxy`就可以根据这个接口来对你
实现代理，也就是说`Proxy`只能代理实现了接口的类。这也就是为什么我们要写一个`Interface`来作为`Service`,然后在里面写一些注解之类的。如果接触过`JAVAEE`的话，
`Spring`里的AOP动态代理是采用`cglib`来修改字节码实现的动态代理，而且不需要实现接口，感兴趣的朋友可以看一下。回到这里，就是通过`Proxy`这个类利用反射，对你写的接口进行解析
获取到你申明的方法，然后对你的方法实现框架想要实现的逻辑，来完成所谓的ä»£理。
我们来看代码。

- 第一步，就是检验你定义的`service`接口是不是正确。简单看下代码,首先如果不是接口会抛出异常，还有为了避免出现bug,和保证API都是统一的标准，不允许定义的`Service`接口继承别的接口
```java
static <T> void validateServiceInterface(Class<T> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }
    // Prevent API interfaces from extending other interfaces. This not only avoids a bug in
    // Android (http://b.android.com/58753) but it forces composition of API declarations which is
    // the recommended pattern.
    if (service.getInterfaces().length > 0) {
      throw new IllegalArgumentException("API interfaces must not extend other interfaces.");
    }
  }
  ```
- 第二步，如果你在前面`creat()`的时候，设置过`validateEagerly`为`true`的话，它会在这一步将所有的你`Service`中声明的`Method`在这里都å始化了,并且缓存起来
```java
private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method)) {
        loadServiceMethod(method);
      }
    }
  }

  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
  ```
  这里在解析你用`annotations`标注的`Method`时也一样用到了`Builder`这种模式，通过`ServiceMethod`这个类来解析你的标注，将标准转化为实际的逻辑。
  这里面的代码比较多，我就不再贴了，其实里面的逻辑比较单一，但是比较复杂。主要就是根据不同的标注，来生成对应的对象，你用着有多简单就有框架来给你承受多复杂。只看一下他的`Constructor`
  最后会得到着一些东西。
  ```java
    ServiceMethod(Builder<R, T> builder) {
      this.callFactory = builder.retrofit.callFactory();
      this.callAdapter = builder.callAdapter;
      this.baseUrl = builder.retrofit.baseUrl();
      this.responseConverter = builder.responseConverter;
      this.httpMethod = builder.httpMethod;
      this.relativeUrl = builder.relativeUrl;
      this.headers = builder.headers;
      this.contentType = builder.contentType;
      this.hasBody = builder.hasBody;
      this.isFormEncoded = builder.isFormEncoded;
      this.isMultipart = builder.isMultipart;
      this.parameterHandlers = builder.parameterHandlers
    }
  ```

- 第三步，这时候就进入到每个方法的代理实现里来了。实际上这里面是已经进入了上面例子中的第三句了，因为是为每一个其中的方法实ç°代理`Call<List<Integer>> call = gitHub.contributors("xx", "xx");`的流程了。
如果是`Object`声明的方法，直接执行原方法，然后`platform.isDefaultMethod(method)`在`Android`平台直接返回`false`，所以这里直接忽略。
- 第四步，这里如果第二步没有build过这个方法，或者缓存里没有会`build`这个方法，缓存里有的话直接取过来。
- 第五步，根据`serviceMethod`初始化`OkHttpCall`,真正执行请求是交给这个类来执行的。
- 第六步，根据`OkHttpCall`最后返回`CallAdapter`适配后的你想要的类型.到这里就通过代理得到了一个所有参数，`headers`或者其他都准备好了的，并且也通过`CallAdapter`实现了返回数据包装的一个完整的数据类型.

讲到这里，准备工作都已经做齐了，就等着最后执行了。这里的`Call`是根据你设置的`CallAdapter`来返回的，比如如果你熟悉`Rxjava`，那结合`Rxjava`，这里也可以
返回一个`Observable`.当然你å¨定义这个`Service`接口的时候也应该声明为这个返回类型。就算是`Call` ,也不是返回`OkHttpCall`,前面讲到了`ExecutorCallbackCall`来静态代理了
`OkHttpCall`，实际上这里返回的是`ExecutorCallbackCall`.

### 执行

如果是`ExecutorCallbackCall`的话，提供了同步的`excute`和异步的`enqueue`来执行这个请求，并且提供一个`Callback`回调的接口来处理调用成功
或者失败。调用之后是如何拿到数据之后，被`Converter`转化，被`CallAdapter`包装然后返回给我们的呢？
来我们慢慢分析。前面我们提到了，其实所有的请求执行，实际上都是`OkHttpCall`这个类在操作。`OkHttpCall`实现了`Call`接口，就是一些请求的常用逻辑，同步异步cancel等等，
不管是同步还是异步，最后都是拿到返回的`Response`转换成我们想要的数据。我们挑一个`OkHttpCall`中同步的方法看看：

```java 
  @Override 
  public Response<T> execute() throws IOException {
      okhttp3.Call call;

      synchronized (this) {
        ... 中间逻辑很简单就省略了

      return parseResponse(call.execute());
     }
  }


 Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
  ```
首先我们就来看`Retrofit`在执行后是怎么讲`response`转换成我们想要的数据的。`excute()`执行后中间有点失败取消的逻辑，最后就是直接把成功后的`response`交给
`parseResponse()`这个方法，这里先转化为一个没有`body`数据的`response`来做状态判断，如果需要转换数据，把原来的`ResponseBody`转换为一个静态代理的`ExceptionCatchingRequestBody`
交给`serviceMethod.toResponse(catchingBody)`，主要是为了做一些异常处理。顺着这个流程我们进`ServiceMethod`来看看`toResponse（）`这个方法。
```
/** Builds a method return value from an HTTP response body. */
 public ServiceMethod build() {
   ...
    responseConverter = createResponseConverter();
    ...
 }

  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }

 

private Converter<ResponseBody, T> createResponseConverter() {
      Annotation[] annotations = method.getAnnotations();
      try {
        return retrofit.responseBodyConverter(responseType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create converter for %s", responseType);
      }
    }
 ```

很简单就是交给了`Converter`来做转换。`Converter`看起来是不是很眼熟。前面我们好像设置了啊。最后又回到了`Retrofit`这个类，来看看
`responseBodyConverter（）`这个方法：
```java
  public <T> Converter<T, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
    return nextRequestBodyConverter(null, type, parameterAnnotations, methodAnnotations);
  }

  public <T> Converter<T, RequestBody> nextRequestBodyConverter(Converter.Factory skipPast,
      Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
    checkNotNull(type, "type == null");
    checkNotNull(parameterAnnotations, "parameterAnnotations == null");
    checkNotNull(methodAnnotations, "methodAnnotations == null");

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter.Factory factory = converterFactories.get(i);
      Converter<?, RequestBody> converter =
          factory.requestBodyConverter(type, parameterAnnotations, methodAnnotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<T, RequestBody>) converter;
      }
    }

   ...build string
    throw new IllegalArgumentException(builder.toString());
  }
  ```
  其实很简单，就是返回`factory.requestBodyConverter(type, parameterAnnotations, methodAnnotations, this);`，就是å·¥厂造一个`Converter`
  这个工厂造的`Converter`怎么造，框架是不管的，总之你按照我给你定义的标准造一个来就是了。感兴趣就去看看`'retrofit-converters'`这个包里是怎么造的，也很简单
  然后通过`Converter`的`convert()`方法就把你想要的类型数据返回给你了，这个`convert()`方法也是你在实现`Converter`要自己实现的，当然源码里提供了一些实现，你自己去看。


  
  整个流程就是这样的。希望对你阅读源代码有帮助。
