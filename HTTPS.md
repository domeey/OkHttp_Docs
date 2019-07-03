# HTTPS

OkHttp尽力平衡两个对立的关注点：

- **Connectivity** ：尽可能多的连接到不同的主机。包括使用最新版本 [boringssl](https://boringssl.googlesource.com/boringssl/) 的先进的主机，也包括那些使用老版本[OpenSSL](https://www.openssl.org/)的相对落后的主机。
- **Security** ：链接的安全。包括远端服务使用证书验证和数据交换的加密。

在和一个HTTPS服务协商时，OkHttp需要知道使用哪个版本的TLS和 [cipher suites](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-cipher-suite/)。客户端想要最大的连接，要包括过时的TLS版本和低强度的加密算法。想要更安全的客户端需要使用最新的TLS版本和最强的加密算法。

具体的安全和连接决策是由 [ConnectionSpec](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-connection-spec/) 来实现的。OkHttp包括四种内置的链接规则：

- `RESTRICTED_TLS` 是安全的配置，是为了满足最严格的需求。
- `MODERN_TLS` 是安全的配置，可以链接到现代的HTTPS。
- `COMPATIBLE_TLS` 是安全的配置，可以链接到安全但不是现代HTTPS的服务。
- `CLEARTEXT` 是不安全的配置，主要用来链接 `http://` 。

这些大致上是根据 [Google Cloud Policies](https://cloud.google.com/load-balancing/docs/ssl-policies-concepts) 提出的模型设计的。我们会跟踪这个规定的变化。

默认的，OkHttp会尝试`MODERN_TLS` 链接。但是，如果这个模式失败，通过配置客户端的connectionSpecs，你可以降级到 `COMPATIBLE_TLS`链接。

```
OkHttpClient client = new OkHttpClient.Builder()
    .connectionSpecs(Arrays.asList(ConnectionSpec.MODERN_TLS, ConnectionSpec.COMPATIBLE_TLS))
    .build();
```

每个规则的TLS版本和加密算法会随着每次发布的所变更。比如，OkHttp 2.2中，我们为了应对 [POODLE](http://googleonlinesecurity.blogspot.ca/2014/10/this-poodle-bites-exploiting-ssl-30.html) 攻击，不再支持SSL3.0。在OkHttp 2.3 我们不再支持 [RC4](http://en.wikipedia.org/wiki/RC4#Security)。和你的桌面浏览器一样，使用最新的OkHttp是最安全的。

你也可以使用一套自定义的TLS版本号和加密算法来实现你自己的链接规则。比如，这个配置限制使用三种高级的加密算法。缺点是需要使用Android 5.0+ 和一个现在的服务。

```
ConnectionSpec spec = new ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
    .tlsVersions(TlsVersion.TLS_1_2)
    .cipherSuites(
          CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
          CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
          CipherSuite.TLS_DHE_RSA_WITH_AES_128_GCM_SHA256)
    .build();

OkHttpClient client = new OkHttpClient.Builder()
    .connectionSpecs(Collections.singletonList(spec))
    .build();
```

### [Certificate Pinning](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/CertificatePinning.java)

默认的，OkHttp信任主机平台的证书。这个策略可以最大化链接，但是受限于安全认证中心攻击，比如 [2011 DigiNotar attack](http://www.computerworld.com/article/2510951/cybercrime-hacking/hackers-spied-on-300-000-iranians-using-fake-google-certificate.html)。它也假定你的HTTPS服务证书是由安全认证中心颁发的。

使用 [CertificatePinner](http://square.github.io/okhttp/4.x/okhttp/okhttp3/-certificate-pinner/) 来限制哪些证书和哪些安全认证中心是可信的。证书锁定提升了安全性，但是限制了服务团队更新他们TLS证书的能力。不要在没有告知服务TLS领导就使用证书锁定。

```
  public CertificatePinning() {
    client = new OkHttpClient.Builder()
        .certificatePinner(new CertificatePinner.Builder()
            .add("publicobject.com", "sha256/afwiKY3RxoMmLkuRW1l7QsPZTJPwDS2pdDROQjXw8ig=")
            .build())
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://publicobject.com/robots.txt")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    for (Certificate certificate : response.handshake().peerCertificates()) {
      System.out.println(CertificatePinner.pin(certificate));
    }
  }
```

### [Customizing Trusted Certificates](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/okhttp3/recipes/CustomTrust.java)

下边的例子演示了如何替换平台的安全认证中心成自己的。不要在没有通知服务TLS领导前就自定义证书。

```
  private final OkHttpClient client;

  public CustomTrust() {
    SSLContext sslContext = sslContextForTrustedCertificates(trustedCertificatesInputStream());
    client = new OkHttpClient.Builder()
        .sslSocketFactory(sslContext.getSocketFactory())
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://publicobject.com/helloworld.txt")
        .build();

    Response response = client.newCall(request).execute();
    System.out.println(response.body().string());
  }

  private InputStream trustedCertificatesInputStream() {
    ... // Full source omitted. See sample.
  }

  public SSLContext sslContextForTrustedCertificates(InputStream in) {
    ... // Full source omitted. See sample.
  }
```

