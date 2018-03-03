>拦截器调用顺序

![](https://chenyoyoy.github.io/image//2017/03/拦截器的调用顺序.png)

>client.interceptors

这是一个很靠前的拦截器，能够拿取到最原始的请求数据，以及最终的返回结果

使用场景可以有：添加一个请求计数器，统计到所有的请求个数，或者打印所有的请求的输入，输出数据

>RetryAndFollowUpInterceptor

![发起一个普通的网络请求](https://chenyoyoy.github.io/image//2017/03/WechatIMG11.jpeg)

处理逻辑：

* 用一个while(true)的循环作为多次请求的发起者，如果循环次数超过MAX_FOLLOW_UPS=20的值，就抛弃重复次数过多的异常，标志请求失败
* followUpRequest(Response userResponse)方法，根据请求的返回结果，处理重定向，代理授权，服务器授权的情况，然后重新生成request，发起调用流程

	    switch (responseCode) {
	    //需要代理服务器授权
      	 case HTTP_PROXY_AUTH:
         Proxy selectedProxy = route != null
            ? route.proxy()
            : client.proxy();
         if (selectedProxy.type() != Proxy.Type.HTTP) {
          throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
        }
        return client.proxyAuthenticator().authenticate(route, userResponse);
        //需要服务器授权
     	 case HTTP_UNAUTHORIZED:
        return client.authenticator().authenticate(route, userResponse);

        //处理重定向的情况
     	 case HTTP_PERM_REDIRECT:
     	 case HTTP_TEMP_REDIRECT:
        if (!method.equals("GET") && !method.equals("HEAD")) {
          return null;
        }
      	case HTTP_MULT_CHOICE:
     	case HTTP_MOVED_PERM:
    	case HTTP_MOVED_TEMP:
        case HTTP_SEE_OTHER:
        //不允许重定向，即返回null
        if (!client.followRedirects()) return null;
    
        //根据response的location参数，重新生成一个request，用于重试
        String location = userResponse.header("Location");
        if (location == null) return null;
        HttpUrl url = userResponse.request().url().resolve(location);
    
        // Don't follow redirects to unsupported protocols.
        if (url == null) return null;
    
        // If configured, don't follow redirects between SSL and non-SSL.
        boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
        if (!sameScheme && !client.followSslRedirects()) return null;
    
        // Most redirects don't include a request body.
        Request.Builder requestBuilder = userResponse.request().newBuilder();
        if (HttpMethod.permitsRequestBody(method)) {
          final boolean maintainBody = HttpMethod.redirectsWithBody(method);
          if (HttpMethod.redirectsToGet(method)) {
            requestBuilder.method("GET", null);
          } else {
            RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
            requestBuilder.method(method, requestBody);
          }
          if (!maintainBody) {
            requestBuilder.removeHeader("Transfer-Encoding");
            requestBuilder.removeHeader("Content-Length");
            requestBuilder.removeHeader("Content-Type");
          }
        }
    
        if (!sameConnection(userResponse, url)) {
          requestBuilder.removeHeader("Authorization");
        }
    
        return requestBuilder.url(url).build();


>BridgeInterceptor

![](https://chenyoyoy.github.io/image//2017/03/BridgeInterceptor.png)

主要工作：

* 根据数据类型，数据长度，在header中填入相关字段；
* 根据url，在header中填入host字段；
* 允许连接keep-alive
* 告诉服务端接受gzip压缩编码
* 加载缓存客户端的Cookie字段，填入到header；
* 设置user-agent 字段
* 获取到数据后，解析gzip的压缩字段，解析 header 和 body ，生成response对象返回

> CacheInterceptor

![](https://chenyoyoy.github.io/image//2017/03/CacheInterceptor.png)

流程：

* 首先从缓存池中取出缓存数据，解析缓存相关字段，生成缓存候选对象，里面包含有效缓存对象以及添加缓存控制字段的网络请求
* 如果缓存结果和网络请求都为空，表示请求失败
* 如果缓存结果有效，返回缓存
* 如果缓存无效，发起网络请求
* 请求到数据后，缓存请求结果，用于下次请求的使用

缓存字段规则：

* Expires/Cache-Control Header是控制浏览器是否直接从浏览器缓存取数据还是重新发请求到服务器取数据。只是Cache-Control比Expires可以控制的多一些， 而且Cache-Control会重写Expires的规则。

* Last-Modified/If-Modified-Since和ETag/If-None-Match是浏览器发送请求到服务器后判断文件是否 已经修改过，如果没有修改过就只发送一个304回给浏览器，告诉浏览器直接从自己本地的缓存取数据；如果修改过那就整个数据重新发给浏览器。

>ConnectInterceptor

![](https://chenyoyoy.github.io/image//2017/03/ConnectInterceptor.png)

核心功能就是获得一个传输通道，用于传输数据；

找到一个可用连接后，生成对应的解码器，因为不同的http协议里面字段不同，需要区分解析

![](https://chenyoyoy.github.io/image//2017/03/newStream.png)

StreamAllocation.findConnection：

     //从缓存池中找到了缓存，返回可用连接
     Internal.instance.get(connectionPool, address, this);
      if (connection != null) {
        return connection;
      }
      
      //省略。。。。
      
      synchronized (connectionPool) {
      route = selectedRoute;
      refusedStreamCount = 0;
      //生成一个新的连接
      result = new RealConnection(connectionPool, selectedRoute);
      acquire(result);
      if (canceled) throw new IOException("Canceled");
    }
    
      result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    routeDatabase().connected(result.route());

  RealConnection.connect中：

  	 try {
        if (route.requiresTunnel()) {
          //如果需要代理，需要在代理上面建立连接通道
          connectTunnel(connectTimeout, readTimeout, writeTimeout);
        } else {
          //不需要代理 直连服务器
          connectSocket(connectTimeout, readTimeout);
        }
        //建立连接协议
        establishProtocol(connectionSpecSelector);
        break;
      } 
      
    private void establishProtocol(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    //不支持https ，直接返回
    if (route.address().sslSocketFactory() == null) {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
      return;
    }
    
    //tls安全连接的建立
    connectTls(connectionSpecSelector);
    
    // http2协议的支持
    if (protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
      http2Connection = new Http2Connection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .listener(this)
          .build();
      http2Connection.start();
    }
  }     


>CallServerInterceptor

![](https://chenyoyoy.github.io/image//2017/03/CallServerInterceptor.png)

核心能力：
构建一个请求的body，然后往服务器写请求数据，然后读取服务器数据；  
生成response，通过调用链一直往上return 。
