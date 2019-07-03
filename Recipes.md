# Recipes

我们已经写了一些例子来演示如何解决一些OkHttp的常见问题。仔细阅读来学习它是怎么使用的。可以自由地复制这些例子。

### Synchronous Get

下载一个文件，打印响应头并且以String来输出响应体。

响应体的 `string()` 方法对于小文件来说非常方便和高效。但是如果响应体非常大（大于1MiB），请不要使用 `string()` ，因为它会把整个文件加载到内存中。这种情况，最好以流的方式来读取响应体。

```
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://publicobject.com/helloworld.txt")
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      Headers responseHeaders = response.headers();
      for (int i = 0; i < responseHeaders.size(); i++) {
        System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
      }

      System.out.println(response.body().string());
    }
  }
```

### Asynchronous Get

在工作线程下载一个文件，然后当响应返回时，会得到一个回调。回调会在响应头好的时候开始。读取响应体可能还是阻塞的。OkHttp通常不提供异步API来分块获取响应体。

```
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    client.newCall(request).enqueue(new Callback() {
      @Override public void onFailure(Call call, IOException e) {
        e.printStackTrace();
      }

      @Override public void onResponse(Call call, Response response) throws IOException {
        try (ResponseBody responseBody = response.body()) {
          if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

          Headers responseHeaders = response.headers();
          for (int i = 0, size = responseHeaders.size(); i < size; i++) {
            System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
          }

          System.out.println(responseBody.string());
        }
      }
    });
  }
```

### Access Headers

典型的HTTP头像一个 `Map<String, String>`：每个字段有一个value或没有。但一些headers可以有多个值，比如Guava’s [Multimap](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Multimap.html)。例如，对HTTP响应来说，提供多个头是常见合法的。OkHttp的API尽量让两种方式都非常方便。

处理请求头时，使用 `header(name, value)` 来设置那些唯一的name。如果已经存在了values，他们会在新值添加前被移除。使用 `addHeader(name, value)` 可以在不移除已经存在的values前提下添加一个header。

读取响应的header时，使用 `header(name)` 可以返回对应的最后一个value。通常也是唯一的。如果没有值，会返回null。如果想要读取一个字段的所有值，使用 `headers(name)`。

使用 `Headers` 以索引方式可以访问所有的头。

```
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://api.github.com/repos/square/okhttp/issues")
        .header("User-Agent", "OkHttp Headers.java")
        .addHeader("Accept", "application/json; q=0.5")
        .addHeader("Accept", "application/vnd.github.v3+json")
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println("Server: " + response.header("Server"));
      System.out.println("Date: " + response.header("Date"));
      System.out.println("Vary: " + response.headers("Vary"));
    }
  }
```

### Posting a String

使用HTTP POST来发送一个请求体。这个例子post一个markdown文件到一个渲染服务器，这个服务可以把markdown渲染成HTML。因为整个请求体是同时在内存中加载，所以避免使用这个API来post大文件（大于1MiB）。

```
  public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    String postBody = ""
        + "Releases\n"
        + "--------\n"
        + "\n"
        + " * _1.0_ May 6, 2013\n"
        + " * _1.1_ June 15, 2013\n"
        + " * _1.2_ August 11, 2013\n";

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, postBody))
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println(response.body().string());
    }
  }
```

### [Post Streaming](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/PostStreaming.java)

现在我们来以流的方式POST请求体。请求体是在写的同时生成的。这个流直接写到了Okio的缓存池。你的程序可能想使用 `OutputStream`，可以通过`BufferedSink.outputStream()`来获取。

```
  public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    RequestBody requestBody = new RequestBody() {
      @Override public MediaType contentType() {
        return MEDIA_TYPE_MARKDOWN;
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        sink.writeUtf8("Numbers\n");
        sink.writeUtf8("-------\n");
        for (int i = 2; i <= 997; i++) {
          sink.writeUtf8(String.format(" * %s = %s\n", i, factor(i)));
        }
      }

      private String factor(int n) {
        for (int i = 2; i < n; i++) {
          int x = n / i;
          if (x * i == n) return factor(x) + " × " + i;
        }
        return Integer.toString(n);
      }
    };

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(requestBody)
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println(response.body().string());
    }
  }
```

### [Posting a File](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/PostFile.java)

把文件作为请求体非常简单。

```
  public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    File file = new File("README.md");

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, file))
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println(response.body().string());
    }
  }
```

### [Posting form parameters](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/PostForm.java)

使用 `FormBody.Builder` 来创建一个像HTML的 `<form>` 标签的请求体。键和值都会使用HTML兼容格式的URL加密。

```
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    RequestBody formBody = new FormBody.Builder()
        .add("search", "Jurassic Park")
        .build();
    Request request = new Request.Builder()
        .url("https://en.wikipedia.org/w/index.php")
        .post(formBody)
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println(response.body().string());
    }
  }
```

### [Posting a multipart request](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/PostMultipart.java)

`MultipartBody.Builder` 可以创建复杂的请求体来兼容HTML上传文件的格式。多媒体请求体的每个部分都是一个请求体，可以定义自己的header。如果设置，这些headers应该是来描述这部分的请求体，比如它的`Content-Disposition`。 `Content-Length` 和 `Content-Type` headers会在可用时自动添加。

```
  /**
   * The imgur client ID for OkHttp recipes. If you're using imgur for anything other than running
   * these examples, please request your own client ID! https://api.imgur.com/oauth2
   */
  private static final String IMGUR_CLIENT_ID = "...";
  private static final MediaType MEDIA_TYPE_PNG = MediaType.parse("image/png");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    // Use the imgur image upload API as documented at https://api.imgur.com/endpoints/image
    RequestBody requestBody = new MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("title", "Square Logo")
        .addFormDataPart("image", "logo-square.png",
            RequestBody.create(MEDIA_TYPE_PNG, new File("website/static/logo-square.png")))
        .build();

    Request request = new Request.Builder()
        .header("Authorization", "Client-ID " + IMGUR_CLIENT_ID)
        .url("https://api.imgur.com/3/image")
        .post(requestBody)
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println(response.body().string());
    }
  }
```

### [Parse a JSON Response With Moshi](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/ParseResponseWithMoshi.java)

[Moshi](https://github.com/square/moshi) 是一个JSON和Java对象转换的简单库。我们使用它来解压转换一个JSON响应。

请注意： 在解压响应体时，`ResponseBody.charStream()` 使用响应头的 `Content-Type` 来选择字符集。如果没有指定，使用默认的 `UTF-8` 。

```
  private final OkHttpClient client = new OkHttpClient();
  private final Moshi moshi = new Moshi.Builder().build();
  private final JsonAdapter<Gist> gistJsonAdapter = moshi.adapter(Gist.class);

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://api.github.com/gists/c2a7c39532239ff261be")
        .build();
    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      Gist gist = gistJsonAdapter.fromJson(response.body().source());

      for (Map.Entry<String, GistFile> entry : gist.files.entrySet()) {
        System.out.println(entry.getKey());
        System.out.println(entry.getValue().content);
      }
    }
  }

  static class Gist {
    Map<String, GistFile> files;
  }

  static class GistFile {
    String content;
  }
```

### [Response Caching](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/CacheResponse.java)

为了缓存响应，你需要一个可读可写的缓存目录，并且要限制它的大小。这个目录应该是私有的，不被信任的应用不能读取。

一个缓存目录同时有多个缓存是错误的。多数应用应该只调用一次 `new OkHttpClient()` ，然后配置它的缓存，最后在其它地方全部使用这个单例。否则，两个缓存实例会相互覆盖，破坏响应缓存，最终可能导致程序崩溃。

响应缓存使用HTTP headers来做所有的配置。可以添加像`Cache-Control: max-stale=3600` 的请求头，OkHttp缓存会使用它们。你的服务会通过它的像`Cache-Control: max-age=9600`的响应头来配置响应会被缓存多长时间。也有一些缓存的header可以强制使用响应缓存，强制使用网络响应，或者强制使用通过虚拟的GET来保证的合法的网络响应。

```
  private final OkHttpClient client;

  public CacheResponse(File cacheDirectory) throws Exception {
    int cacheSize = 10 * 1024 * 1024; // 10 MiB
    Cache cache = new Cache(cacheDirectory, cacheSize);

    client = new OkHttpClient.Builder()
        .cache(cache)
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    String response1Body;
    try (Response response1 = client.newCall(request).execute()) {
      if (!response1.isSuccessful()) throw new IOException("Unexpected code " + response1);

      response1Body = response1.body().string();
      System.out.println("Response 1 response:          " + response1);
      System.out.println("Response 1 cache response:    " + response1.cacheResponse());
      System.out.println("Response 1 network response:  " + response1.networkResponse());
    }

    String response2Body;
    try (Response response2 = client.newCall(request).execute()) {
      if (!response2.isSuccessful()) throw new IOException("Unexpected code " + response2);

      response2Body = response2.body().string();
      System.out.println("Response 2 response:          " + response2);
      System.out.println("Response 2 cache response:    " + response2.cacheResponse());
      System.out.println("Response 2 network response:  " + response2.networkResponse());
    }

    System.out.println("Response 2 equals Response 1? " + response1Body.equals(response2Body));
  }
```

为了避免响应使用缓存，可以用 [`CacheControl.FORCE_NETWORK`](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-cache-control/-f-o-r-c-e_-n-e-t-w-o-r-k/)。为了避免响应使用网络 ，可以用 [`CacheControl.FORCE_CACHE`](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-cache-control/-f-o-r-c-e_-c-a-c-h-e/)。要注意：如果使用 `FORCE_CACHE`，但是响应需要网络，OkHttp会返回 `504 Unsatisfiable Request`的响应。

### [Canceling a Call](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/CancelCall.java)

使用 `Call.cancel()` 可以立刻中止一个正在进行的请求。如果线程正在写一个请求或者读一个响应，它会收到 `IOException`。使用这个可以在请求不需要时保护网络；比如，当用户从一个应用离开时。不论是同步还是异步请求都可以取消。

```
  private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
        .build();

    final long startNanos = System.nanoTime();
    final Call call = client.newCall(request);

    // Schedule a job to cancel the call in 1 second.
    executor.schedule(new Runnable() {
      @Override public void run() {
        System.out.printf("%.2f Canceling call.%n", (System.nanoTime() - startNanos) / 1e9f);
        call.cancel();
        System.out.printf("%.2f Canceled call.%n", (System.nanoTime() - startNanos) / 1e9f);
      }
    }, 1, TimeUnit.SECONDS);

    System.out.printf("%.2f Executing call.%n", (System.nanoTime() - startNanos) / 1e9f);
    try (Response response = call.execute()) {
      System.out.printf("%.2f Call was expected to fail, but completed: %s%n",
          (System.nanoTime() - startNanos) / 1e9f, response);
    } catch (IOException e) {
      System.out.printf("%.2f Call failed as expected: %s%n",
          (System.nanoTime() - startNanos) / 1e9f, e);
    }
  }
```

### [Timeouts](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/ConfigureTimeouts.java)

当对方不可达时，使用timeouts可以让请求失败而停止。网络中断可能是因为客户端链接问题，服务端不可用问题，或者其它两者间的任何问题。OkHttp支持链接，读和写的超时。

```
  private final OkHttpClient client;

  public ConfigureTimeouts() throws Exception {
    client = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .writeTimeout(10, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
        .build();

    try (Response response = client.newCall(request).execute()) {
      System.out.println("Response completed: " + response);
    }
  }
```

### [Per-call Configuration](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/PerCallSettings.java)

所有的HTTP客户端设置全部都在OkHttpClient中，包括代理设置，超时设置和缓存设置。当你需要为一个请求改变设置时，调用`OkHttpClient.newBuilder()`。这个方法返回一个和原来client共用同一个链接池，同一个dispatcher和同样样配置的builder。下面的例子中，我们一个请求500ms超时，另一个3000ms超时。

```
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/1") // This URL is served with a 1 second delay.
        .build();

    // Copy to customize OkHttp for this request.
    OkHttpClient client1 = client.newBuilder()
        .readTimeout(500, TimeUnit.MILLISECONDS)
        .build();
    try (Response response = client1.newCall(request).execute()) {
      System.out.println("Response 1 succeeded: " + response);
    } catch (IOException e) {
      System.out.println("Response 1 failed: " + e);
    }

    // Copy to customize OkHttp for this request.
    OkHttpClient client2 = client.newBuilder()
        .readTimeout(3000, TimeUnit.MILLISECONDS)
        .build();
    try (Response response = client2.newCall(request).execute()) {
      System.out.println("Response 2 succeeded: " + response);
    } catch (IOException e) {
      System.out.println("Response 2 failed: " + e);
    }
  }
```

### [Handling authentication](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/Authenticate.java)

OkHttp可以自动重试未验证的请求。当响应是 `401 Not Authorized`，验证器会被要求提供证书。验证器的实现应该创建一个带有证书的新的请求。如果没有证书，返回null来跳过重试。

使用 `Response.challenges()` 来得到身份验证的策略和边界。如果是完成一个 `Basic` 的验证，使用 `Credentials.basic(username, password)`来加密请求头。

```
  private final OkHttpClient client;

  public Authenticate() {
    client = new OkHttpClient.Builder()
        .authenticator(new Authenticator() {
          @Override public Request authenticate(Route route, Response response) throws IOException {
            if (response.request().header("Authorization") != null) {
              return null; // Give up, we've already attempted to authenticate.
            }

            System.out.println("Authenticating for response: " + response);
            System.out.println("Challenges: " + response.challenges());
            String credential = Credentials.basic("jesse", "password1");
            return response.request().newBuilder()
                .header("Authorization", credential)
                .build();
          }
        })
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/secrets/hellosecret.txt")
        .build();

    try (Response response = client.newCall(request).execute()) {
      if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

      System.out.println(response.body().string());
    }
  }
```

为了避免当验证不可用时重试太多，你可以返回null来放弃。比如，你可能想在这些证书已经被尝试时跳过重试：

```
  if (credential.equals(response.request().header("Authorization"))) {
    return null; // If we already failed with these credentials, don't retry.
   }
```

当你遇到应用定义好的限制次数时，跳过重试：

```
  if (responseCount(response) >= 3) {
    return null; // If we've failed 3 times, give up.
  }
```

上边的这些代码依赖于 `responseCount()` 方法：

```
  private int responseCount(Response response) {
    int result = 1;
    while ((response = response.priorResponse()) != null) {
      result++;
    }
    return result;
  }
```