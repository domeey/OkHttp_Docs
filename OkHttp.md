# OkHttp

HTTP是现代网络应用的根本，是我们交换数据和媒体的方式。高效的HTTP可以让我们的产品加载更快并且节省带宽。

OkHttp是一个天生就高效的HTTP客户端：

- HTTP/2支持所有指向同一host的请求共享一个socket
- 连接池可以减少请求延迟（如果HTTP/2不可用）
- 透明的GZIP压缩缩小了下载体积
- 响应缓存完全避免重复的请求网络

当网络异常时，OkHttp可以保存现场，在常见的网络问题解决后，可以安静地自动恢复。如果我们的服务有多个IP地址，OkHttp会在第一个失败后尝试其它的。对于IPv4+IPv6模式和服务部署在多数据中心来说，这是必要的。OkHttp支持现代的TLS（TLS 1.3，ALPN，certificate pinning）。对于许多链接，它可以配置去回退。

使用OkHttp非常简单。请求和响应的API是使用链式的builder来设计的。同时支持同步阻塞请求和异步回调请求。

### Get a URL

下面的例子是请求一个URL，然后把返回值以String输出：

```
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

### Post to a Server

下面的例子是向一个服务post数据：

```
public static final MediaType JSON
    = MediaType.get("application/json; charset=utf-8");

OkHttpClient client = new OkHttpClient();

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(JSON, json);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

更多例子[OkHttp Recipes page](http://square.github.io/okhttp/recipes/).

### Requirements

OkHttp需要 Android 5.0+（API Level 21+）和 Java 8+。

OkHttp依赖 [Okio](https://github.com/square/okio)，Okio是一个高性能I/O的库 。

我们强烈推荐保持OkHttp的更新。因为随着浏览器的自动更新，使用HTTPS的客户端对于潜在的安全问题是非常重要的防护手段。我们跟踪TLS生态的变化，调用OkHttp来提高链接的性能和安全。

OkHttp使用平台内置的TLS实现。在Java平台，OkHttp也支持和Java集成BoringSSL的[Conscrypt](https://github.com/google/conscrypt/)。如果Conscrypt是第一个安全提供者，OkHttp就会使用它。

```
Security.insertProviderAt(Conscrypt.newProvider(), 1);
```

 OkHttp 3.12.x 分支支持Android 2.3+（API Level 9+）和 Java 7+。这些平台不支持TLS 1.2 ，不应该使用。但是因为升级困难，我们会在2020年底之前继续修复关键的bug到3.12.x分支。

### Releases

我们的 [change log](http://square.github.io/okhttp/changelog/) 上有发布历史。

```
implementation("com.squareup.okhttp3:okhttp:4.0.0")
```

### R8 / ProGuard

如果使用了R8或 ProGuard，请添加 [`okhttp3.pro`](https://github.com/square/okhttp/blob/master/okhttp/src/main/resources/META-INF/proguard/okhttp3.pro)。

也可能需要Okio的规则。

### MockWebServer

OkHttp 包含了一个测试HTTP，HTTPS和HTTP/2客户端的库。

```
testImplementation("com.squareup.okhttp3:mockwebserver:4.0.0")
```

### License

```
Copyright 2019 Square, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```