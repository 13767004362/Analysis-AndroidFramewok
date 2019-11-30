## **OkHttp v3.11.0 执行过程**


执行一个请求，先从OkHttpClient 中的newCall()开始：


```java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
通过OkHttpClient对象，构建出一个与Request一一对应的RealCall对象。

接下来，查看RealCall的newRealCall():

```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    //为每个Call设置一个事件处理器，默认是EventListener.NONE(未做事件处理)。
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```
若是对请求需要一些特别的，可以自定义EventListener。


接下来，查看RealCall的构造器：

根据上面的默认配置，第三个参数forWebSocket是为false,不使用websocket。

```
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    //默认 false,不使用websocket的。
    this.forWebSocket = forWebSocket;
    // 配置一个重连的拦截器，用于重连机制。
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
```

接下来，看下RealCall的同步执行execute()方法：

```java
  @Override public Response execute() throws IOException {
   
   synchronized (this) {
     // 从这里，可以知道RealCall只能执行一次，不可以重复调用execute()
    if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    
    // 为不同平台，设置用于捕获Call的堆栈，
    // 这里，android平台使用dalvik.system.CloseGuard去记录RealCall的使用情况(详情，查看AndroidPlatform )。
    captureCallStackTrace();
    
    //开始通知EventListener监听器,request开始执行。
    eventListener.callStart(this);
    
    try {
      // 将RealCall添加到Dispatcher中等待同步执行的请求队列中。
      client.dispatcher().executed(this);
      
      //开始，通过一系列的Interceptor拦截器，执行网络请求。
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      // 通知EventListener监听器，调用失败
      eventListener.callFailed(this, e);
      throw e;
    } finally {
       
      client.dispatcher().finished(this);
    }
  }
```

接下来，查看RealCall的getResponseWithInterceptorChain():

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // 开始添加一些列的拦截器
    List<Interceptor> interceptors = new ArrayList<>();
    //先添加OkHttpClient端中配置的通用拦截器
    interceptors.addAll(client.interceptors());
    //开始添加重试的拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //开始添加缓存的拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    //默认forWebSocket为false,执行http请求,会添加OkHttpClient端中配置网络拦截器
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    // 开始构建与RealCall一一对的Chain对象。
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
添加完一系列的拦截器后，开始构建与RealCall一一对的Chain对象。

拦截器调用递增：
>寻常的Interceptor-->重试的RetryAndFollowUpInterceptor-->BridgeInterceptor
>
>-->缓存的CacheInterceptor-->ConnectInterceptor-->配置的网络Interceptor
>
>-->CallServerInterceptor


接下来查看，RealInterceptorChain中proceed()方法：

从上可以知道，默认情况下，构建RealInterceptorChain对象时，传入的streamAllocation, httpCodec, connection是为空的。

```java
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }
```
接下来，来到重载方法proceed()中：

```java
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
     
    // 传入角标若是大于等于拦截器的个数，抛出异常。
    if (index >= interceptors.size()) throw new AssertionError();
    //开始累积调用次数
    calls++;

    //若是已经执行过Request，但网络拦截器中，重新调用poceed()执行请求情况下，修改过Request	的Url，则抛出异常。
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    //一个Chain对象，只能调用一个网络拦截器执行。
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // 构建一个新的Chain对象，传递到下一个拦截器中。
    // 注意点：这里会逐个调用下一个拦截器，无特殊情况会直到最后一个CallServerInterceptor拦截器，返回结果，类似递归调用，走完全部的拦截器。
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    //获取到最后一个拦截器返回的响应
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    //检验拦截器返回的响应是否为空.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }
    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

接下来，跟着一些列的拦截器，看下request请求在这些拦截器中，做了哪些处理。

由上面的代码可知道：拦截器调用顺序逐个递增。
>寻常的Interceptor-->重试的RetryAndFollowUpInterceptor-->BridgeInterceptor
>
>-->缓存的CacheInterceptor-->ConnectInterceptor-->配置的网络Interceptor
>
>-->CallServerInterceptor


默认寻常的拦截器为空，由上层自由配置(例如：添加通用表头，通用参数或者日志的拦截器)。

因此，先来，看下RetryAndFollowUpInterceptor的intercept():

```
  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();
    // 创建一个对象用于，管理多次的connetion、stream。
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    //上次的请求的响应
    Response priorResponse = null;
    while (true) {
      // 上层调用取消，则结束
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        // 传递到下一个拦截器，执行相应的操作，直到返回最后的响应。
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getFirstConnectException();
        }
        releaseConnection = false;
        //发生该异常，继续重试。
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
         //发生该异常，继续重试。
        continue;
      } finally {
        // 除了正常请求、IOException、RouteException外，发生其他的异常时，releaseConnection会为true,则关闭，释放资源
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      //把上次执行请求返回的Response附带上
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp;
      try {
         // 检查返回的响应，处理一些特殊情况(验证、重定向之类)，便于再次请求。
        followUp = followUpRequest(response, streamAllocation.route());
      } catch (IOException e) {
        streamAllocation.release();
        throw e;
      }
      // 注意点：这里，正常获取到响应结果，结束循环，返回响应。
      if (followUp == null) { 
        if (!forWebSocket) { //若是http请求，则释放资源
          streamAllocation.release();
        }
        return response;
      }
      //关闭响应的body
      closeQuietly(response.body());

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```

从上可以知道：RetryAndFollowUpInterceptor负责重试机制和创建 StreamAllocation对象对象。

接下来，查看BridgeInterceptor的intercept()：

```java
 @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    // 构建一个新的网络请求
    Request.Builder requestBuilder = userRequest.newBuilder();
    RequestBody body = userRequest.body();
    
    // 处理post请求，添加content-type、content-length的表头。
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
    //让网络请求保持keeep-alive
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }
    // 若是没有指定Accept-Encoding编码类型或者非Range分段下载的请求，默认会添加gzip。
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }
    //添加cookie
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }
    //	添加默认的User-Agent
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }
    // 传递给下一个拦截器，直到返回最终的响应。
    Response networkResponse = chain.proceed(requestBuilder.build());
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
    //通过网络响应构建出一个新的用户请求对应的用户响应。
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
    //当网络请求是gzip压缩，且返回网络响应也是指定gzip压缩，并且有响应体时，OkHttp将响应的body通过gzip压缩编码后返回。
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```

从上可知,BridgeInterceptor负责将用户请求，通过添加一系列的header(content-type、cookie、默认gzip压缩),组装成一个网络请求。接下来，将网络请求传递给下一个拦截器，等待最终的网络响应返回。最后，将网络响应重新，组装成用户请求的一一对应的用户响应。

接下来，查看CacheInterceptor 的intercept()：

```
  @Override public Response intercept(Chain chain) throws IOException {
    //缓存中原始的响应数据
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //创建出网络请求对应的缓存策略，用于检查是否使用缓存还是使用网络数据(详情，查看CacheStrategy)。
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    //给上层提供的Cache，追踪缓存策略。
    if (cache != null) {
      cache.trackResponse(strategy);
    }
    // 当在缓存存在请求对应的响应，但不符合缓存策略，存在问题(失效之类)的情况，关闭缓存中响应。
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body());  
    }

    // 当禁止使用网络(即添加了only-if-cached表头)，且并没有缓存，则返回504的标示结果.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    //符合缓存策略，不需要执行网络请求，则返回缓存中的响应
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      // 传递给下一个拦截器，执行网络请求，直到返回最后的网络响应。
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // 执行网络请求过程中，若存在上次缓存中失效的数据，则关闭缓存的响应，防止泄漏。
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // 存在缓存数据，但需要通过网络请求(Get方式)去更新数据。
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();
        // 将最新的网络响应数据，更新到缓存中。
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
    //将网络响应、缓存中响应数据，一起附带上，构建出新的响应数据。
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      // 网络响应存在数据，且符合缓存策略，则将数据写入缓存中。
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          //移除缓存中请求对应的响应数据。
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }

```

从上可知，CacheInterceptor 负责从缓存中获取请求对应的响应，且构建出一个缓存策略。若是响应不符合缓存策略，则发起网络请求，去拉去最新数据，再根据缓存策略判断是否缓存响应。反之，存在可用的响应，则不需要使用网络请求，则返回缓存中的响应。

接下来，查看ConnectInterceptor的intercept()：

```
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // 需要确保网络请求的安全，但存在一种特殊的get方式请求(存在本地缓存的响应数据，但get方式去查看是否需要更新)
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // 从连接池中查找可用的connection。
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();
    // 传递给下一个拦截器，执行网络请求，直到返回响应。
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
从上可知，ConnectInterceptor负责筛选可以用的 RealConnection对象。

接下来，执行网络请求之前，会先传递到上层提供的网络拦截器，做特殊处理后，最后来到CallServerInterceptor。

最后，看下CallServerInterceptor的intercept()：

```java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    //若是http请求， 这里， HttpCodec是Http2Codec	.
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();
    // 开始写入请求中的表头，通知EventListener监听器。
    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    //开始写入请求的body
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }
    // 完成网络请求的写入操作
    httpCodec.finishRequest();
    //开始构建，响应的Builder对象，读取Header
    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }
    
    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);
   
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
     // 开始构建 响应的body
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }
    // 根据实际上需要，是否关闭StreamAllocation，释放资源
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }
    // 检查网络请求的状态
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }
    // 返回最终的网络响应
    return response;
  }
```

经过一系列的拦截器，最终返回响应到RealCall的` getResponseWithInterceptorChain()`。

