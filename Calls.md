# Calls

HTTP 客户端的工作是接收请求然后产出响应。非常简单，但是实际应用中却非常复杂。

### Requests

每个 HTTP 请求包含一个URL，一个请求方法（像 GET 和 POST），和一组的请求头。

请求也可能包含一个请求体：指定内容类型的一组数据流。

### Responses

响应会在应答请求时带一个响应码（像表示成功的200，找不到的404），响应头，和它自己可选的响应体。

### Rewriting Requests

当使用OkHttp做一个HTTP请求时，你是在一个较高的层次来描述请求：“带着这些请求头把这个URL的内容给我”。为了正确性和性能考虑，OkHttp会在发送你的请求前重写它。

OkHttp可能添加原来请求中缺失的请求头，包括`Content-Length`, `Transfer-Encoding`, `User-Agent`, `Host`, `Connection`, 和`Content-Type`。除非 `Accept-Encoding` 这个请求头已经添加，否则，OkHttp会为了传输响应压缩而添加它。如果你有cookies，OkHttp也会用它们添加一个  `Cookie` 的请求头。

一些请求会有一个缓存的响应。当这个缓存的响应失效时，OkHttp会做一个有虚拟的 GET 请求去下载更新的响应，如果这个响应比缓存的更新。这个功能需要请求头加上 `If-Modified-Since` 和`If-None-Match`。

### Rewriting Responses

如果传输压缩启用了，OkHttp会丢弃对应响应头的`Content-Encoding` 和`Content-Length`，因为它们和解压后的请求体是不一致的。

如果一个虚拟的 GET 请求成功了，网络的响应和本地的缓存会按照规则进行合并。

### Follow-up Requests

当你请求的URL被重定向了，服务器会返回一个类似302的响应码来表明这个文件的新的URL。OkHttp会自动重定向然后去获取最终的响应。

如果响应声明了权限问题，OkHttp会向 [`Authenticator`](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-authenticator/) （如果配置了）去请求授权。如果认证器提供了一个授权，请求会带着这个授权去重试。

### Retrying Requests

有时链接会失败：可能是链接池中的链接坏了或者断开链接了，也可能是服务不可达。如果有一条路径可用，OkHttp会使用其它路由去重试。

### Calls

配合上边的机制，你的一个简单请求可能产生很多请求和响应。不论中间需要多少请求和响应，OkHttp使用 `Call` 处理整个任务来满足你的请求。通常来说，这些发生的次数不是很多。但是，即使是在URL被重定向或者一个可选的IP失效时，你的代码还可用，这是让人放心的。

Calls可以被以下两种中的一种方式执行：

- **Synchronous**：线程被阻塞直到响应可读
- **Asynchronous**：把请求加入一个线程的队列中，在另一个线程响应返回时，通过回调接收

Calls可以在任何线程中取消。如果请求没有完成会导致请求失败。写请求和读响应的那些代码在call被取消时会遇到 `IOException`。

### Dispatch

对于同步的请求来说，你使用自己的线程，所以自己管理请求的并发。过高的请求并发会浪费资源，过低的请求并发会加大延迟。

对于异步请求来说， [`Dispatcher`](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-dispatcher/) 实现了最高请求并发的策略。你可以设置每个服务的最高并发（默认是5），和总共的最高并发（默认是64）。

