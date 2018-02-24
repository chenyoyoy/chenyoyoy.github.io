<blockquote>
  HTTP2 的优势
</blockquote>

<ul>
<li>采用了连接池技术，对于同一服务器的请求，可以使用之前建立的连接，不用每次都建立，降低了延迟；</li>
<li>采用多路复用的原理，在一个connect上面分多个流接收/发送数据，能够有更大的并发能力；</li>
<li>采用了Hpack压缩方式，对请求头字段进行了压缩，降低了数据传输量。</li>
</ul>

<blockquote>
  OkHttp中非常重要的一些组件
</blockquote>

<ul>
<li>HttpUrl 请求资源的实体，解析url里面的结构，形成host ，scheme，port ，pathSegments 等相关的字段</li>
<li>Address 为了拿到服务器资源，在整个连接建立过程中，需要使用到的组件的集合</li>
<li>Dns dns解析器，默认采用系统的 。connect 最终是采用的ip+port 的方式建立socket连接 ，所以需要拿到IP </li>
<li>SocketFactory  socket工厂，用于生成Socket连接Android java 都有各自的实现，也可以自己定制</li>
<li>Authenticator 认证器 ，服务器连接建立的时候，可能需要对客户端进行认真</li>
<li>ProxySelector 代理选择器，默认使用系统的</li>
<li>Proxy 代理，通过代理选择器选中的连接代理</li>
<li>HostnameVerifier 主机名验证器，用于验证建立ssl连接的主机名是否被认可</li>
<li>CertificatePinner 证书指定 ，里面配置一些服务器证书的签名，用于指定服务器的证书校验，防止中间人攻击</li>
<li>StreamAllocation 流分配器，里面包含建立connect前需要的Address,route，ConnectionPool，RealConnection，RouteSelector，HttpCodec</li>
<li>Route 路由，能够触及到服务器资源的路径配置，包括 代理选择 ，ip选择等</li>
<li>HttpCodec 请求解析器。http2 采用多路复用，需要关联StreamId从同一个connect中区分解析不同请求的数据</li>
<li>Connection 对应客户端和服务器建立的socket连接</li>
<li>Http2Connection http2连接，里面实现对Http2连接的控制，以及各个请求申请的流资源管理</li>
</ul>

<blockquote>
  Connect连接的建立：
</blockquote>

RealConnection 是实际连接的建立者，里面包含了底层socket的建立过程：

<pre><code>public void connect(

  while (true) {
  try {
    if (route.requiresTunnel()) {
      //需要走代理，建立代理通道
      connectTunnel(connectTimeout, readTimeout, writeTimeout);
    } else {
      //直接连接服务器
      connectSocket(connectTimeout, readTimeout);
    }
    //根据连接参数，建立连接协议
    establishProtocol(connectionSpecSelector);
    break;
  } catch (IOException e) {
   //...
  }
}
}


private void connectTunnel(int connectTimeout, int readTimeout, int writeTimeout)
  throws IOException {

  //构建一个简单的请求，通过代理发送，保证连接连通性
Request tunnelRequest = createTunnelRequest();
HttpUrl url = tunnelRequest.url();
int attemptedConnections = 0;
int maxAttempts = 21;
while (true) {
  if (++attemptedConnections &gt; maxAttempts) {
    throw new ProtocolException("Too many tunnel connections attempted: " + maxAttempts);
  }

  //建立一个连接，这里肯定是和代理建立的
  connectSocket(connectTimeout, readTimeout);

  //通过代理，发送一个简单的请求，保证通过代理能够和服务器连接上
  tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url);

  if (tunnelRequest == null) break; // Tunnel successfully created.
}
}


private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
Proxy proxy = route.proxy();
Address address = route.address();

//需要代理的时候，就跟代理建立socket，不需要的时候，跟跟服务器直接连接
rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
    ? address.socketFactory().createSocket()
    : new Socket(proxy);

rawSocket.setSoTimeout(readTimeout);
try {
  //建立连接
  Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
} catch (ConnectException e) {
  ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
  ce.initCause(e);
  throw ce;
}
//拿到流操作对象
source = Okio.buffer(Okio.source(rawSocket));
sink = Okio.buffer(Okio.sink(rawSocket));
}

private void establishProtocol(ConnectionSpecSelector connectionSpecSelector) throws IOException {
//不需要建立安全连接，直接使用http 1.1的协议
if (route.address().sslSocketFactory() == null) {
  protocol = Protocol.HTTP_1_1;
  socket = rawSocket;
  return;
}

//在已有的连接上，建立安全协议
connectTls(connectionSpecSelector);

//如果是HTT2的协议，需要初始化http2Connection，对分组流数据实现操作
if (protocol == Protocol.HTTP_2) {
  socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
  http2Connection = new Http2Connection.Builder(true)
      .socket(socket, route.address().url().host(), source, sink)
      .listener(this)
      .build();
  http2Connection.start();
}
}

private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
  Address address = route.address();
  SSLSocketFactory sslSocketFactory = address.sslSocketFactory();
  boolean success = false;
  SSLSocket sslSocket = null;
  try {
  //在socket连接基础上，包装一层SSLSocket
  sslSocket = (SSLSocket) sslSocketFactory.createSocket(
      rawSocket, address.url().host(), address.url().port(), true /* autoClose */);

  //确定加密套件，tls版本，以及其他的一些扩展配置
  ConnectionSpec connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket);
  if (connectionSpec.supportsTlsExtensions()) {
    Platform.get().configureTlsExtensions(
        sslSocket, address.url().host(), address.protocols());
  }

  //强制显示握手，不强制调用这个方法的话，默认的在Socket中写入数据的时候，也会隐式调用   但是现在需要在写入数据前，进行一些判断，所以强制调用一次
  sslSocket.startHandshake();

  //握手后，会和服务器建立一个连接的session
  Handshake unverifiedHandshake = Handshake.get(sslSocket.getSession());

  //验证主机名是否是需要请求资源对应的服务器
  if (!address.hostnameVerifier().verify(address.url().host(), sslSocket.getSession())) {
    X509Certificate cert = (X509Certificate) unverifiedHandshake.peerCertificates().get(0);
    throw new SSLPeerUnverifiedException("Hostname " + address.url().host() + " not verified:"
        + "\n    certificate: " + CertificatePinner.pin(cert)
        + "\n    DN: " + cert.getSubjectDN().getName()
        + "\n    subjectAltNames: " + OkHostnameVerifier.allSubjectAltNames(cert));
  }

  //验证特定的服务器证书，就是具有内置指定签名的证书，保证证书唯一性
  address.certificatePinner().check(address.url().host(),
      unverifiedHandshake.peerCertificates());

  //session建立成功，且验证通过 保存协议信息
  String maybeProtocol = connectionSpec.supportsTlsExtensions()
      ? Platform.get().getSelectedProtocol(sslSocket)
      : null;
  socket = sslSocket;
  source = Okio.buffer(Okio.source(socket));
  sink = Okio.buffer(Okio.sink(socket));
  handshake = unverifiedHandshake;
  protocol = maybeProtocol != null
      ? Protocol.get(maybeProtocol)
      : Protocol.HTTP_1_1;
  success = true;
} catch (AssertionError e) {
  if (Util.isAndroidGetsocknameError(e)) throw new IOException(e);
  throw e;
} finally {
  if (sslSocket != null) {
    Platform.get().afterHandshake(sslSocket);
  }
  if (!success) {
    closeQuietly(sslSocket);
  }
}
}

private Request createTunnel(int readTimeout, int writeTimeout, Request tunnelRequest,
  HttpUrl url) throws IOException {
 String requestLine = "CONNECT " + Util.hostHeader(url, true) + " HTTP/1.1";
while (true) {
  Http1Codec tunnelConnection = new Http1Codec(null, null, source, sink);
  source.timeout().timeout(readTimeout, MILLISECONDS);
  sink.timeout().timeout(writeTimeout, MILLISECONDS);
  tunnelConnection.writeRequest(tunnelRequest.headers(), requestLine);
  tunnelConnection.finishRequest();
  Response response = tunnelConnection.readResponseHeaders(false)
      .request(tunnelRequest)
      .build();
  // The response body from a CONNECT should be empty, but if it is not then we should consume
  // it before proceeding.
  long contentLength = HttpHeaders.contentLength(response);
  if (contentLength == -1L) {
    contentLength = 0L;
  }
  Source body = tunnelConnection.newFixedLengthSource(contentLength);
  Util.skipAll(body, Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
  body.close();

  switch (response.code()) {
    case HTTP_OK:
   //...
      return null;

    //如果代理需要认证客户端身份，调用身份认证器 ，重新构建一个携带token的请求用于建立连接
    case HTTP_PROXY_AUTH:
      tunnelRequest = route.address().proxyAuthenticator().authenticate(route, response);
      if (tunnelRequest == null) throw new IOException("Failed to authenticate with proxy");

      if ("close".equalsIgnoreCase(response.header("Connection"))) {
        return tunnelRequest;
      }
      break;

    default:
      throw new IOException(
          "Unexpected response code for CONNECT: " + response.code());
  }
}
}

//建立一个最简单的请求，通过代理向服务器发送，保证连通
private Request createTunnelRequest() {
return new Request.Builder()
    .url(route.address().url())
    .header("Host", Util.hostHeader(route.address().url(), true))
    .header("Proxy-Connection", "Keep-Alive") // For HTTP/1.0 proxies like Squid.
    .header("User-Agent", Version.userAgent())
    .build();
}
}
</code></pre>

<blockquote>
  HTTP2 协议中流的管控
</blockquote>

和服务器建立一个连接connection，实际上就是一个socket连接了，往socket 里面读写数据，就能完成和服务器的通信了；

HTT2协议要求在一个socket上面，实现并发的请求，那么就需要对每个请求进行识别，每次读和取的时候都能知道对应的数据流是属于哪个请求的，然后对应的返回给业务层。

在初始化一个连接的时候，我们有拿到这个连接对应协议的解析器：

<pre><code>HttpCodec httpCodec =streamAllocation.newStream(client, doExtensiveHealthChecks);
RealConnection connection = streamAllocation.connection();
</code></pre>

httpCodec 会根据版本区分实现对字段的写入以及解析。

<ul>
<li>发起请求，写入请求头数据

<pre><code>//向socket中写入头部数据
httpCodec.writeRequestHeaders(request);

 //对应的实现方法
 @Override public void writeRequestHeaders(Request request) throws IOException {
 if (stream != null) return;
boolean hasRequestBody = request.body() != null;
//将request中带的字段，拼装成http2 header协议的参数
List&lt;Header&gt; requestHeaders = http2HeadersList(request);
//在connect上面分配一个新请求的流
stream = connection.newStream(requestHeaders, hasRequestBody);
//http2的超时都是设置在流上面的
stream.readTimeout().timeout(client.readTimeoutMillis(), TimeUnit.MILLISECONDS);
stream.writeTimeout().timeout(client.writeTimeoutMillis(), TimeUnit.MILLISECONDS);
</code></pre></li>
<li>newStream的关键代码：

<pre><code>synchronized (this) {
if (shutdown) {
  throw new ConnectionShutdownException();
}
//新的唯一性id
streamId = nextStreamId;
nextStreamId += 2;
//新分配一个流
stream = new Http2Stream(streamId, this, outFinished, inFinished, requestHeaders);
flushHeaders = !out || bytesLeftInWriteWindow == 0L || stream.bytesLeftInWriteWindow == 0L;
if (stream.isOpen()) {
  //将流放到map中，connection对承载的流进行管理
  streams.put(streamId, stream);
 }
 }
</code></pre></li>
<li>流数据的写出与读入

<pre><code>//拿到sink对，就可以往服务器写数据了
Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
//给服务器发送数据
request.body().writeTo(bufferedRequestBody);
bufferedRequestBody.close();

//从响应中读取header字段  这里是io阻塞的，直到能够读取到header数据才往下继续走  或者超时
responseBuilder = httpCodec.readResponseHeaders(false);

//读取到header后，继续读取body
   response = response.newBuilder()
  .body(httpCodec.openResponseBody(response))
  .build();
</code></pre>

<img src="http://test.chenyoyo.cn/wp-content/uploads/2017/03/Http2流复用.png" alt="" /></p></li>
<li><p>writer 提供往连接中根据不同请求写入请求数据的能力

<pre><code>void dataFrame(int streamId, byte flags, Buffer buffer, int byteCount) throws IOException {
byte type = TYPE_DATA;
//写入帧头部数据
frameHeader(streamId, byteCount, type, flags);
if (byteCount &gt; 0) {
//写入帧承载的数据内容
sink.write(buffer, byteCount);
 }
}

  public void frameHeader(int streamId, int length, byte type, byte flags) throws IOException {
  if (length &gt; maxFrameSize) {
  throw illegalArgument("FRAME_SIZE_ERROR length &gt; %d: %d", maxFrameSize, length);
  }
  if ((streamId &amp; 0x80000000) != 0) throw illegalArgument("reserved bit set: %s", streamId);
writeMedium(sink, length);
sink.writeByte(type &amp; 0xff);
sink.writeByte(flags &amp; 0xff);
//写入流的id  
sink.writeInt(streamId &amp; 0x7fffffff);
}
</code></pre></li>
<li>ReaderRunnable 提供不断从socket中读取帧数据的能力

<img src="http://test.chenyoyo.cn/wp-content/uploads/2017/03/reader.png" alt="" />

取出帧数据，解析FrameHeader，然后根据承载数据的类型，丢给对应的方法解析数据
<img src="http://test.chenyoyo.cn/wp-content/uploads/2017/03/frame数据的解析.png" alt="" />

比如：解析到帧数据携带的data，就把对应的数据给到请求
<img src="http://test.chenyoyo.cn/wp-content/uploads/2017/03/connection将数据交给stream.png" alt="" />

//HTTP2 stream会把接受到的数据保存到source中<br />
this.source.receive(in, length);

//需要返回结果数据的时候，从source中拿取数据
Source source = new StreamFinishingSource(stream.getSource());
return new RealResponseBody(response.headers(), Okio.buffer(source));

配合之前的调用链，数据也就从最后的Stream，到response，逐级返回到了业务层

<pre><code>    Response response = getResponseWithInterceptorChain();
if (retryAndFollowUpInterceptor.isCanceled()) {
  signalledCallback = true;
  responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
} else {
  signalledCallback = true;
  responseCallback.onResponse(RealCall.this, response);
}  
</code></pre></li>
</ul>
