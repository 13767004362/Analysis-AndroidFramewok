## **OkHttp的连接池机制**

OkHttp创建address对应的RealConnection过程是在ConnectionInterceptor的intercept()，先从这里看起：

```java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    
    //开始寻找可用的Connection，若是没有可复用的，则创建新的。
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

接下来，先看下StreamAllocation的newStream():

```java
  public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    // 获取网络请求的配置参数
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      //开始寻找可用的RealConnection
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      // 构建出Http协议对应的
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);
      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

接下来，查看findHealthyConnection():

```java
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      //若是上面步骤，创建出新的connection，则不需要安全检查，直接可以使用。
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

接下来，来看findConnection():

```java
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");
      // 若是存在一个已经分配使用过的Connection，需要检查该Connection是否受限，从而创建新的Stream。 
      // 若是第一次请求，则跳过这些。
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }
      // 若是StreamAllocation中不存在已经分配的Connection，则从连接池中获取。
      if (result == null) {
        // Attempt to get a connection from the pool.
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else { // 若是不存在address对应的connection，则记录route
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);
    //分支走向：若是存在可用的RealConnection则，通知 EventListener监听器，则返回
    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      return result;
    }

    //若是需要一个route，则从一系列中的Route列表中获取。默认，第一次执行，route为空，selectedRoute也为空。
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");
      //若是存在一个新的address，则从连接池中去获取是佛可用的RealConnection。
      if (newRouteSelection) {  
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }
      //若是新的address，没有从连接池中找到合适的RealConnection，则立马创建一个新的RealConnection。
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }
        //标记新创建connection对应的route
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    //建立socket连接，进行TCP + TLS 操作. 
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    //记录已经建立连接的Route。
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      //标记，已经存在RealConnection
      reportedAcquired = true;

      // 将RealConnection添加到连接池中，方便后续复用。
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);
    // 通知监听器，已经获取到RealConnection。
    eventListener.connectionAcquired(call, result);
    return result;
  }
```

来看下连接池ConnectionPool中

获取可用的RealConnection:

```java
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      // 根据address和route比对，获取到合适的RealConnection。
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    若是，连接池中没有匹配到合适，则返回空。
    return null;
  }
```
缓存可用的RealConnection的put():

```
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      // 开启一个单线程，清楚过期的连接。
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```




