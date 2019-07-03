# Connections

虽然你只是提供了URL，OkHttp会使用三种：URL，Address，和 Route 来安排和你服务器之间的链接。

### URLs

URLs（如 `https://github.com/square/okhttp`）对于HTTP和网络来说是非常根本的。除了是网络上的全球资源定位功能之外，它们还指定了如何获取资源。

URLs是抽象的：

- 它们指定了请求是简单的 (`http`) 还是加密的 (`https`)，但不会指定加密算法。也不会指定如何去验证证书或者哪个证书是可信的。
- 它们不指定是否使用代理服务或者代理服务器如何验证。

它们也是具体的：每个URL都表明了一个确定的路径（像 `/square/okhttp`）和请求（像 `?q=sharks&lang=en`）。每个服务器有很多URLs。

### Addresses

Addresses指定了一个服务器（像 `github.com`）和所有其它需要链接到这些服务器的东西：端口，HTTPS设置，和 协议类型（像 HTTP/2 和 SPDY）。

同一个地址的URLs 可能共用一些底层的TCP链接。共用底层链接有大量的性能提升：低延迟，高并发（因为 TCP 启动慢）和 节省电量。OkHttp使用一个链接池来自动重用 HTTP/1.x链接和HTTP/2及SPDY链接的多路传输。

OkHttp中，地址的一些字段来自于URL（scheme，hostname，port），其它来自于 OkHttpClient。

### Routes

Routes提供链接到一个服务必需的动态信息。它是一个指定的IP地址（通过DNS查询得到的），准确的代理服务器（如果使用了 [ProxySelector](http://developer.android.com/reference/java/net/ProxySelector.html) ），和 使用哪个版本的TLS去协商（对于HTTPS链接来说）。

对一个地址来说可能有多条路径。比如，在多个数据中心都部署了一个服务可能在DNS响应中产生多个IP地址。

### Connections

当你使用OkHttp请求一个URL时，下面是主要流程：

1. 它使用URL和配置的OkHttpClient来创建一个address。这个地址指定了我们怎样去链接服务。
2. 从链接池中尝试取对应地址的链接。
3. 如果找不到链接，选择一个route去尝试。通常是先做一个DNS请求来得到IP地址，然后选择一个TLS版本，必要时还需要确定一个代理服务器。
4. 如果是一个新的路径，要么创建一个直接的Socket链接，一个TLS通道（HTTPS使用HTTP代理），要么创建一个直接的TLS链接。必要时进行TLS握手。
5. 发送HTTP请求，读取响应。

如果链接有问题，OkHttp会选择其它路径重试。这样OkHttp可以在一些服务地址不可达时进行恢复。当链接池中的链接损坏或者TLS版本不支持时，也是非常有用的。

一旦收到响应，链接就会被回收到链接池中以便可以重复利用。链接不活跃一段时间后会从链接池中移除。

