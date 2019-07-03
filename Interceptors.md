# Interceptors

Interceptors 是一个强大的机制，它可以用来监控，重写和重试请求。下边的例子打印了请求和响应的日志。

```
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```

对任何一个拦截器来说，调用 `chain.proceed(request)` 是非常重要的。所有的HTTP流程，产生一个请求对应的响应都是在这个简单的方法中。

拦截器可以是链式的。假设你有一个压缩拦截器和一个检验的拦截器：你需要决定数据是先压缩还是先检验。OkHttp使用list来跟踪拦截器，拦截器会按照顺序依次调用。

![img](Interceptors.assets/interceptors@2x.png)

### Application Interceptors

拦截器的注册有两种方式：

1. 注册到应用。
2. 注册网络拦截器。

我们会用上边定义的`LoggingInterceptor` 来演示它们的区别。

注册应用拦截器是在 `OkHttpClient.Builder`上调用 `addInterceptor()` ：

```
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

 `http://www.publicobject.com/helloworld.txt` 重定向到了`https://publicobject.com/helloworld.txt`，OkHttp自动跟踪了这次重定向。我们的应用拦截器被调用了一次，从`chain.proceed()`中得到的响应是重定向后的响应：

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on null
User-Agent: OkHttp Example

INFO: Received response for https://publicobject.com/helloworld.txt in 1179.7ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

 `response.request().url()` 和`request.url()`不同，所以我们被重定向了。这两行日志打印了不同的URLs。

### Network Interceptors

注册一个网络拦截器非常类似。调用 `addNetworkInterceptor()` ：

```
OkHttpClient client = new OkHttpClient.Builder()
    .addNetworkInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

当我们运行代码时，这个拦截器运行了两次。一次是请求`http://www.publicobject.com/helloworld.txt`，另一次是重定向的`https://publicobject.com/helloworld.txt`。

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on Connection{www.publicobject.com:80, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=none protocol=http/1.1}
User-Agent: OkHttp Example
Host: www.publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for http://www.publicobject.com/helloworld.txt in 115.6ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/html
Content-Length: 193
Connection: keep-alive
Location: https://publicobject.com/helloworld.txt

INFO: Sending request https://publicobject.com/helloworld.txt on Connection{publicobject.com:443, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA protocol=http/1.1}
User-Agent: OkHttp Example
Host: publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for https://publicobject.com/helloworld.txt in 80.9ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

网络拦截器也包含了更多的数据，比如 OkHttp为了响应支持压缩而添加的`Accept-Encoding: gzip` 。网络拦截器的`Chain` 有一个非空的`Connection` ，它可以用来询问连接服务使用的IP地址和TLS配置。

### Choosing between application and network interceptors

每个拦截器链都有优点。

##### **Application interceptors**

- 不需要关心中间的响应，如重定向和重试。
- 只调用一次，即使HTTP响应是从缓存中来的。
- 关注应用的原始意图。不关心其它像`If-None-Match`这样的OkHttp注入的请求头。
- 允许短路，不调用 `Chain.proceed()`。
- 允许重试然后多次调用 `Chain.proceed()`。

##### **Network Interceptors**

- 可以操作中间的响应，比如重定向和重试。
- 如果是使用缓存响应来短路请求，不会调用。
- 恰好按照数据的流向来跟踪数据。
- 可以访问处理请求的 `Connection` 。

### Rewriting Requests

拦截器可以添加，移除或替换请求头。如果有请求体，拦截器也可以变换请求体。比如，如果能确定服务支持压缩，我们可以使用应用拦截器来添加请求体压缩。

```
/** This interceptor compresses the HTTP request body. Many webservers can't handle this! */
final class GzipRequestInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request originalRequest = chain.request();
    if (originalRequest.body() == null || originalRequest.header("Content-Encoding") != null) {
      return chain.proceed(originalRequest);
    }

    Request compressedRequest = originalRequest.newBuilder()
        .header("Content-Encoding", "gzip")
        .method(originalRequest.method(), gzip(originalRequest.body()))
        .build();
    return chain.proceed(compressedRequest);
  }

  private RequestBody gzip(final RequestBody body) {
    return new RequestBody() {
      @Override public MediaType contentType() {
        return body.contentType();
      }

      @Override public long contentLength() {
        return -1; // We don't know the compressed length in advance!
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        BufferedSink gzipSink = Okio.buffer(new GzipSink(sink));
        body.writeTo(gzipSink);
        gzipSink.close();
      }
    };
  }
}
```

### Rewriting Responses

对应地，拦截器也可以重写响应头和变换响应体。这比重写请求头更加危险，因为它违反了服务器的期望。

如果你的问题比较棘手，也愿意承担后果，重写响应头是非常强大的武器。比如，你可以修改服务侧错误配置的 `Cache-Control` 响应头来达到更好的响应缓存：

```
/** Dangerous interceptor that rewrites the server's cache-control header. */
private static final Interceptor REWRITE_CACHE_CONTROL_INTERCEPTOR = new Interceptor() {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Response originalResponse = chain.proceed(chain.request());
    return originalResponse.newBuilder()
        .header("Cache-Control", "max-age=60")
        .build();
  }
};
```

这种方法对服务侧的修复补充是非常好用的。