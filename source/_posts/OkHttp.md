---
title: OkHttp3 源码浅析
date: 2016-12-06 16:20:52
tags: [Android]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/200.jpg 
---

# 背景

之前的底层网络库基本就是Apache HttpClient和HttpURLConnection。由于HttClient比较难用，官方在Android2.3以后就不建议用了，并且在Android5.0以后废弃了HttpClient，在Android6.0更是删除了HttpClient。

HttpURLConnection是一种多用途、轻量极的HTTP客户端，使用它来进行HTTP操作可以适用于大多数的应用程序，但是在Android 2.2版本之前存在一些bug，所以官方建议在Android2.3以后替代HttpClient，Volley就是按版本分区使用这两个网络库。

然而随着开源届扛把子Square的崛起，OkHttp的开源，这两个网络库只能被淹没在历史洪流中。Android4.4以后HttpURLConnection的底层已经替换成OkHttp实现。OkHttp配合同样是Square开源的Retrofit，网络请求变得更简便，功能更强大。

# OkHttp

OkHttp是一个现代，快速，高效的网络库，OkHttp 库的设计和实现的首要目标是高效。

 - 支持 HTTP/2和SPDY，这使得对同一个主机发出的所有请求都可以共享相同的套接字连接；
 - 如果 HTTP/2和SPDY不可用，OkHttp会使用连接池来复用连接以提高效率。
 - 支持Gzip降低传输内容的大小
 - 支持Http缓存
 - 会从很多常用的连接问题中自动恢复。如果服务器配置了多个IP地址，OkHttp 会自动重试一个主机的多个 IP 地址。
 - 使用Okio来大大简化数据的访问与存储，提高性能

## 简单使用

简单的异步请求

```java
    OkHttpClient client = new OkHttpClient();
    Request request = new Request.Builder()
            .url(url)
            .build();

    client.newCall(request).enqueue(new Callback() {
        public void onFailure(Request request, IOException e)  {
        
    }

        public void onResponse(Response response) throws IOException {
            System.out.println(response.body().string());
        }
});
```

使用非常的简答，发送请求，拿到异步结果。

## OkHttpClient

跟下源码，OkHttpClient.newCall实现

```java
public class OkHttpClient implements Cloneable, Call.Factory{
  public static final class Builder {
    Dispatcher dispatcher;
    Proxy proxy;
    List<Protocol> protocols;
    List<ConnectionSpec> connectionSpecs;
    final List<Interceptor> interceptors = new ArrayList<>();
    final List<Interceptor> networkInterceptors = new ArrayList<>();
    ProxySelector proxySelector;
    CookieJar cookieJar;
    Cache cache;
    InternalCache internalCache;
    SocketFactory socketFactory;
    SSLSocketFactory sslSocketFactory;
    CertificateChainCleaner certificateChainCleaner;
    HostnameVerifier hostnameVerifier;
    CertificatePinner certificatePinner;
    Authenticator proxyAuthenticator;
    Authenticator authenticator;
    ConnectionPool connectionPool;
    Dns dns;
    boolean followSslRedirects;
    boolean followRedirects;
    boolean retryOnConnectionFailure;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;
    }
    ...
    ...
    @Override public Call newCall(Request request) {
    return new RealCall(this, request);
    ...
    ...
  }
}
```
OkHttpClient通过Builder实例化，实现了Call.Factory接口创建了一个RealCall的实例，而RealCall是Call接口的实现。

```java
public interface Call {
  Request request();
  Response execute() throws IOException;
  void enqueue(Callback responseCallback);
  void cancel();
  boolean isExecuted();
  boolean isCanceled();
  interface Factory {
    Call newCall(Request request);
  }
}
```

## RealCall

RealCall中封装了OKHttpClient和Request

```java
  protected RealCall(OkHttpClient client, Request originalRequest) {
    this.client = client;
    this.originalRequest = originalRequest;
  }
  
  @Override public void enqueue(Callback responseCallback) {
    enqueue(responseCallback, false);
  }

  void enqueue(Callback responseCallback, boolean forWebSocket) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
  }
  
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    private final boolean forWebSocket;

    private AsyncCall(Callback responseCallback, boolean forWebSocket) {
      super("OkHttp %s", redactedUrl().toString());
      this.responseCallback = responseCallback;
      this.forWebSocket = forWebSocket;
    }
    ...

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain(forWebSocket);
        if (canceled) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        //注意这一句代码
        client.dispatcher().finished(this);
      }
    }
  }
```
调用enqueue封装成AsyncCall交给OKHttpClient的dispatcher线程池执行。

## Dispatcher线程池

OkHttp的dispatcher参数是直接new出来的。先看下enqueue方法，将AsyncCall当做参数传递进来

```java
public final class Dispatcher {
  /** 最大并发请求数为64 */
  private int maxRequests = 64;
  /** 每个主机最大请求数为5 */
  private int maxRequestsPerHost = 5;

  /** 线程池 */
  private ExecutorService executorService;

  /** 准备执行的请求 */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** 正在执行的异步请求，包含已经取消但未执行完的请求 */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** 正在执行的同步请求，包含已经取消单未执行完的请求 */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
  
    public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
}
```

构造一个线程池ExecutorService：

```java
executorService = new ThreadPoolExecutor(
    0, //corePoolSize 最小并发线程数,如果是0的话，空闲一段时间后所有线程将全部被销毁。
    Integer.MAX_VALUE, //maximumPoolSize: 最大线程数，当任务进来时可以扩充的线程最大值，当大于了这个值就会根据丢弃处理机制来处理
    60, //keepAliveTime: 当线程数大于corePoolSize时，多余的空闲线程的最大存活时间
    TimeUnit.SECONDS,//单位秒
    new SynchronousQueue<Runnable>(),//工作队列,先进先出        Util.threadFactory("OkHttp Dispatcher", false));//单个线程的工厂
```

构建了一个最大线程数为Integer.MAX_VALUE的线程池，也就是说，是个不设最大上限的线程池（其实有限制64个），有多少任务添加进来就新建多少线程，以保证I/O任务中高阻塞低占用的过程中，不会长时间卡在阻塞上。当工作完成后，线程池会在60s内相继关闭所有线程。

还记得刚才在AsyncCall.execute() finally中的内容吗

```
finally {
    client.dispatcher().finished(this);
  }
  ...
  
  /** Used by {@code AsyncCall#run} to signal completion. */
  synchronized void finished(AsyncCall call) {
    if (!runningAsyncCalls.remove(call)) throw new AssertionError("AsyncCall wasn't running!");
    promoteCalls();
  }
  

  //Dispatcher.java
  private void promoteCalls() {
  //超过阈值 返回
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```

当AsyncCall执行完成后，调用Disptcher的finish()方法，调用promoteCalls()方法，如果超过阈值，继续等待，否则取出缓存区的任务执行，顺序是先进先出。

Dispatcher线程池总结

 - 调度线程池Disptcher实现了高并发，低阻塞的实现
 - 采用Deque作为缓存，先进先出的顺序执行
 - 任务在try/finally中调用了finished函数，控制任务队列的执行顺序，而不是采用锁，减少了编码复杂性提高性能

## Interceptor

调度基本整明白了，AsyncCall 中的execute具体内容还没有分析，主要就一行代码。

```java
    @Override protected void execute() {
    boolean signalledCallback = false;
      try {
        ...
        Response response = getResponseWithInterceptorChain(forWebSocket);
        ...
      } finally {
        client.dispatcher().finished(this);
      }
    }
    
    private Response getResponseWithInterceptorChain(boolean     forWebSocket) throws IOException {
    Interceptor.Chain chain = new         ApplicationInterceptorChain(0, originalRequest, forWebSocket);
    return chain.proceed(originalRequest);
    }

```

从方法名字基本可以猜到是干嘛的，调用``chain.proceed(originalRequest);``将request传递进来，从拦截器链里拿到返回结果。那么拦截器Interceptor是干嘛的，Chain是干嘛的呢？继续往下看ApplicationInterceptorChain

```java
class ApplicationInterceptorChain implements Interceptor.Chain {
    private final int index;
    private final Request request;

    ApplicationInterceptorChain(int index, Request request, boolean forWebSocket) {
      this.index = index;
      this.request = request;
      this.forWebSocket = forWebSocket;
    }

    @Override public Connection connection() {
      return null;
    }

    @Override public Request request() {
      return request;
    }

    @Override public Response proceed(Request request) throws IOException {
      // If there's another interceptor in the chain, call that.
      if (index < client.interceptors().size()) {
        Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
        Interceptor interceptor = client.interceptors().get(index);
        Response interceptedResponse = interceptor.intercept(chain);

        if (interceptedResponse == null) {
          throw new NullPointerException("application interceptor " + interceptor
              + " returned null");
        }

        return interceptedResponse;
      }

      // No more interceptors. Do HTTP.
      return getResponse(request, forWebSocket);
    }
  }
```

ApplicationInterceptorChain实现了Interceptor.Chain接口，持有Request的引用。

```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```

proceed方法中判断index（此时为0）是否小于client.interceptors(List<Interceptor> )的大小，如果小于也就是说client.interceptors还有Interceptor，那么就再封装一个ApplicationInterceptorChain，只不过index + 1，然后取出第index个Interceptor将chain传递进去。传递进去干嘛呢？我们看一个用法，以实际项目为例

```java
HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor(new RetrofitLogger());
        interceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(interceptor)
        .retryOnConnectionFailure(true)
        .connectTimeout(15, TimeUnit.SECONDS)
        .addInterceptor(getCommonParameterInterceptor())
        .addNetworkInterceptor(getTokenInterceptor())
        .build();
        
@Override
protected Interceptor getCommonParameterInterceptor() {
    return new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request originalRequest = chain.request();
            Request request = originalRequest;
            if (!originalRequest.method().equalsIgnoreCase("POST")) {
                HttpUrl modifiedUrl = originalRequest.url().newBuilder()
                        .addQueryParameter("version_code", String.valueOf(AppUtils.getVersionCode()))
                        .addQueryParameter("app_key", "nicepro")
                        .addQueryParameter("app_device", "Android")
                        .addQueryParameter("app_version", AppUtils.getVersionName())
                        .addQueryParameter("token", AccountUtils.getToken())
                        .build();
                request = originalRequest.newBuilder().url(modifiedUrl).build();
            }
            return chain.proceed(request);
        }
    };
}

@Override
protected Interceptor getTokenInterceptor() {
    return new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request originalRequest = chain.request();
            Request authorised = originalRequest.newBuilder()
                    .header("app-key", "nicepro")
                    .header("app-device", "Android")
                    .header("app-version", AppUtils.getVersionName())
                    .header("os", AppUtils.getOs())
                    .header("os-version", AppUtils.getAndroidVersion() + "")
                    .header("Accept", "application/json")
                    .header("User-Agent", "Android/retrofit")
                    .header("token", AccountUtils.getToken())
                    .build();
            return chain.proceed(authorised);
        }
    };
}
```

可以看到每个Interceptor的intercept方法中做了一些操作后，最后都会调用``chain.proceed(request)``方法，而这个chain就是每次prceed方法中生成的ApplicationInterceptorChain，用index+1的方式递归调用OkHttClient中的Interceptors，进行拦截操作，比如可以用来监控log，修改请求，修改结果，供开发者自定义参数添加等等，然后最终调用的还是最初的index=0的那个chain的proceed方法中的``getResponse(request, forWebSocket);``。

可以说OkHttp是用chain串联起拦截器，而每个拦截器都有能力返回Response，返回Response即终止整个调用链，这种设计模式称为[责任链模式](https://zh.wikipedia.org/wiki/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F)。这种模式为OkHttp提供了强大的装配能力，极大的提高了OkHttp的扩展性和可维护性。

在Android系统中最典型的责任链模式就是View的Touch传递机制，一层一层传递直到被消费。

官方的一张图就能很好的解释Interceptor
![](https://raw.githubusercontent.com/wiki/square/okhttp/interceptors@2x.png)
整个流程很清晰。这种设计真是太棒了，值得学习！

## 连接池复用

我们知道进行一次tcp网络请求，一般要三次握手连接，四次握手断开连接。一次完整的http请求过程见下图。

![](http://7xs23g.com1.z0.glb.clouddn.com/http.jpg)

如果请求重复的地址，那么重复的连接和断开连接就成了延长整个时间的的重要因素，特别是在复杂的网络环境下，每次请求传输数据的大小将不再是请求速度的决定性因素。

http有一种``keepalive connections``的机制，可以在传输后仍然保持连接，当客户端需要再次获取数据时，直接使用刚刚空闲下来的连接而不需要再次握手。

Okhttp支持5个并发KeepAlive，默认链路生命为5分钟(链路空闲后，保持存活的时间)，关于OkHttp连接池复用详细请看这篇文章 [OkHttp3源码分析[复用连接池]](http://www.jianshu.com/p/92a61357164b)。

## DNS解析

对比上一张图的一次完整的Http请求，在复杂的天朝网络环境下，相信大多数开发者都碰到过很奇怪的网络问题，比如运营商动态插入辣鸡html代码嵌入广告，比如运营商缓存请求数据导致用户请求到的数据不是最新的问题，比如某些运营商只支持``put\post``请求，而不支持``delete``请求，比如运营商。。。这些问题大部分都跟DNS相关。

为了解决DNS劫持的问题，我们在薄荷app上做了很多优化工作，比如使用HTTP DNS（我们使用的DNSPod）代替系统自带的libc库去查询运营商的DNS服务器，直接拿到IP地址进行IP直连，其中又做了一些缓存和选择最优IP的一些操作。解决掉了很大一部分用户反馈的网络问题。

而在OkHttp中，可以直接配置DNS，默认是系统自带的``Dns.SYSTEM``。

```java
      // Try each address for best behavior in mixed IPv4/IPv6 environments.
      List<InetAddress> addresses = address.dns().lookup(socketHost);
      for (int i = 0, size = addresses.size(); i < size; i++) {
        InetAddress inetAddress = addresses.get(i);
        inetSocketAddresses.add(new InetSocketAddress(inetAddress, socketPort));
      }
```

注意结果是数组，即一个域名可能会有多个IP，如果一个IP不通，会自动重连下一个IP。

开发者就可以新建一个Dns类复写``lookup``方法通过HTTP DNS请求IP地址，其中新建一个``HttpDNSClient``来请求DNS，插入拦截器来配置缓存时间，容错处理等等，然后在构建OkHttpClient时加入``dns``方法即可。

```java
client = new OkHttpClient.Builder().addNetworkInterceptor(getLogger())
        .dispatcher(getDispatcher())
        //配置DNS查询实现
        .dns(HTTP_DNS)
        .build();
```

这样的全局HTTP DNS解析真是足够简单高效，并且完全是无侵入性的，丝毫不影响正常的网络请求。

# 总结

本文基本讲了下OkHttp3的大概流程，Interceptor的基本原理，DNS的可选配置等。涉及到socket和Okio流相关的都没有讲到，有兴趣的读者可以在参考文章自行搜索。总结来说，OkHttp基本可以满足日常开发的需求，并且性能足够强大，配合Retrofit + Rxjava更是效率翻倍。如果你在开发新的项目，强烈建议你扔掉Volley，拥抱Retrofit。

# 参考

 - [OkHttp3源码分析[综述]](http://www.jianshu.com/p/aad5aacd79bf)
 - [OkHttp3应用[HTTP DNS的实现]](http://www.jianshu.com/p/9803a6efb672)
 - [从OKHttp框架看代码设计](https://gold.xitu.io/post/581311cabf22ec0068826aff)
 - [拆轮子系列：拆 OkHttp](http://blog.piasy.com/2016/07/11/Understand-OkHttp/)
 - [一次完整的Http请求过程](http://blog.csdn.net/liudong8510/article/details/7908093)




