# OkHttp 源码分析

## get请求案例


### 1. 请求的demo

 	private static void testSyn(String url) {
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url("http://www.baidu.com").build();
        Call call = okHttpClient.newCall(request);
        try {
            Response execute = call.execute();
            System.out.println("before get");
            System.out.println(execute.body().string());
            System.out.println("after get");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    
    
    private static void testAsync(String url) {
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url("http://www.baidu.com").build();
        Call call = okHttpClient.newCall(request);
        System.out.println("before post");
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                System.out.println("exception:"+e.toString());
                System.out.println("after post");
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                System.out.println("content: "+response.body().string());
                System.out.println("after post");
            }
        });
    }
    

 
### 2. new OkHttpClient()

 构建了相关的请求参数,门面模式

### 3. Request.Build()

 请求参数的构建
 
### 4. OkHttpClient.newCall(request)

 构建了一个请求
 
	  /**
	   * Prepares the {@code request} to be executed at some point in the future.
	   */
	  @Override public Call newCall(Request request) {
	    return RealCall.newRealCall(this, request, false /* for web socket */);
	  }
	 
	 
### 5. 同步执行

	 @Override public Response execute() throws IOException {
	    synchronized (this) {
	      if (executed) throw new IllegalStateException("Already Executed");
	      executed = true;
	    }
	    captureCallStackTrace();
	    eventListener.callStart(this);
	    try {
	      client.dispatcher().executed(this);
	      Response result = getResponseWithInterceptorChain();
	      if (result == null) throw new IOException("Canceled");
	      return result;
	    } catch (IOException e) {
	      eventListener.callFailed(this, e);
	      throw e;
	    } finally {
	      client.dispatcher().finished(this);
	    }
	  }
	  
1. 如果在执行中,抛出异常
2. 记录为已经执行
3. 记录调用堆栈
4. 通知开始执行
5. 调用dispatcher 开始执行executed(realCall)
6. 构建所有的责任链模型处理器队列,处理数据
7. 通知client.dispatcher(realCall) 处理完成
7. 返回处理的结果

#### 5.1 Dispatcher的executed(realCall)


	  /** Used by {@code Call#execute} to signal it is in-flight. */
	  synchronized void executed(RealCall call) {
	    runningSyncCalls.add(call);
	  }

只是放到了执行的队列中


####5.2  RealCall.getResponseWithInterceptorChain()

	 Response getResponseWithInterceptorChain() throws IOException {
	    // Build a full stack of interceptors.
	    List<Interceptor> interceptors = new ArrayList<>();
	    interceptors.addAll(client.interceptors());
	    interceptors.add(retryAndFollowUpInterceptor);
	    interceptors.add(new BridgeInterceptor(client.cookieJar()));
	    interceptors.add(new CacheInterceptor(client.internalCache()));
	    interceptors.add(new ConnectInterceptor(client));
	    if (!forWebSocket) {
	      interceptors.addAll(client.networkInterceptors());
	    }
	    interceptors.add(new CallServerInterceptor(forWebSocket));
	
	    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
	        originalRequest, this, eventListener, client.connectTimeoutMillis(),
	        client.readTimeoutMillis(), client.writeTimeoutMillis());
	
	    return chain.proceed(originalRequest);
	  }
	  


处理器过程

1. OkHttpClient的前置处理器(默认无)
2. RetryAndFollowUpInterceptor
3. BridgeInterceptor
4. CacheInterceptor
5. ConnectInterceptor
6. 非webSocket下的网络拦截器
7. CallServerInterceptor
8. 构建了责任链的数据 : RealInterceptorChain, 将处理器以及相关数据,传递过去
9. 调用责任链开始处理

##### 5.2.1 RealInterceptorChain的核心逻辑


	  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
	      RealConnection connection) throws IOException {
	    if (index >= interceptors.size()) throw new AssertionError();
	
	    calls++;
	
	    // If we already have a stream, confirm that the incoming request will use it.
	    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
	      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
	          + " must retain the same host and port");
	    }
	
	    // If we already have a stream, confirm that this is the only call to chain.proceed().
	    if (this.httpCodec != null && calls > 1) {
	      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
	          + " must call proceed() exactly once");
	    }
	
	    // Call the next interceptor in the chain.
	    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
	        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
	        writeTimeout);
	    Interceptor interceptor = interceptors.get(index);
	    Response response = interceptor.intercept(next);
	
	    // Confirm that the next interceptor made its required call to chain.proceed().
	    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
	      throw new IllegalStateException("network interceptor " + interceptor
	          + " must call proceed() exactly once");
	    }
	
	    // Confirm that the intercepted response isn't null.
	    if (response == null) {
	      throw new NullPointerException("interceptor " + interceptor + " returned null");
	    }
	
	    if (response.body() == null) {
	      throw new IllegalStateException(
	          "interceptor " + interceptor + " returned a response with no body");
	    }
	
	    return response;
	  }
	  
1. 参数校验
2. 构建下一个  RealInterceptorChain
3. 获取当前的interceptor,调用interceptor的intercept方法



##### 5.2.2. RetryAndFollowUpInterceptor的处理逻辑


	  @Override public Response intercept(Chain chain) throws IOException {
	    Request request = chain.request();
	    RealInterceptorChain realChain = (RealInterceptorChain) chain;
	    Call call = realChain.call();
	    EventListener eventListener = realChain.eventListener();
	
	    streamAllocation = new StreamAllocation(client.connectionPool(), createAddress(request.url()),
	        call, eventListener, callStackTrace);
	
	    int followUpCount = 0;
	    Response priorResponse = null;
	    while (true) {
	      if (canceled) {
	        streamAllocation.release();
	        throw new IOException("Canceled");
	      }
	
	      Response response;
	      boolean releaseConnection = true;
	      try {
	        response = realChain.proceed(request, streamAllocation, null, null);
	        releaseConnection = false;
	      } catch (RouteException e) {
	        // The attempt to connect via a route failed. The request will not have been sent.
	        if (!recover(e.getLastConnectException(), false, request)) {
	          throw e.getLastConnectException();
	        }
	        releaseConnection = false;
	        continue;
	      } catch (IOException e) {
	        // An attempt to communicate with a server failed. The request may have been sent.
	        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
	        if (!recover(e, requestSendStarted, request)) throw e;
	        releaseConnection = false;
	        continue;
	      } finally {
	        // We're throwing an unchecked exception. Release any resources.
	        if (releaseConnection) {
	          streamAllocation.streamFailed(null);
	          streamAllocation.release();
	        }
	      }
	
	      // Attach the prior response if it exists. Such responses never have a body.
	      if (priorResponse != null) {
	        response = response.newBuilder()
	            .priorResponse(priorResponse.newBuilder()
	                    .body(null)
	                    .build())
	            .build();
	      }
	
	      Request followUp = followUpRequest(response);
	
	      if (followUp == null) {
	        if (!forWebSocket) {
	          streamAllocation.release();
	        }
	        return response;
	      }
	
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
	      } else if (streamAllocation.codec() != null) {
	        throw new IllegalStateException("Closing the body of " + response
	            + " didn't close its backing stream. Bad interceptor?");
	      }
	
	      request = followUp;
	      priorResponse = response;
	    }
	  }
	  
todo: 待分析

网络请求重试等逻辑,在此处



##### 5.2.3. BridgeInterceptor的处理逻辑

	
	  @Override public Response intercept(Chain chain) throws IOException {
	    Request userRequest = chain.request();
	    Request.Builder requestBuilder = userRequest.newBuilder();
	
	    RequestBody body = userRequest.body();
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
	
	    if (userRequest.header("Connection") == null) {
	      requestBuilder.header("Connection", "Keep-Alive");
	    }
	
	    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
	    // the transfer stream.
	    boolean transparentGzip = false;
	    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
	      transparentGzip = true;
	      requestBuilder.header("Accept-Encoding", "gzip");
	    }
	
	    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
	    if (!cookies.isEmpty()) {
	      requestBuilder.header("Cookie", cookieHeader(cookies));
	    }
	
	    if (userRequest.header("User-Agent") == null) {
	      requestBuilder.header("User-Agent", Version.userAgent());
	    }
	
	    Response networkResponse = chain.proceed(requestBuilder.build());
	
	    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
	
	    Response.Builder responseBuilder = networkResponse.newBuilder()
	        .request(userRequest);
	
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
	  
将应用层的逻辑转换为实际的网络层的代码

1. 请求体,content-type,长度,gzip等
2. 交给下一个处理体,处理内容
3. 将返回的结果,进行处理
4. 存储cookie
5. 构建response,并且设置请求为当前的请求(todo: 细节待分析)
6. 如果是gzip压缩,重新设置response的body
7. 返回response

##### 5.2.3. CacheInterceptor的处理逻辑

	
	 @Override public Response intercept(Chain chain) throws IOException {
	    Response cacheCandidate = cache != null
	        ? cache.get(chain.request())
	        : null;
	
	    long now = System.currentTimeMillis();
	
	    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
	    Request networkRequest = strategy.networkRequest;
	    Response cacheResponse = strategy.cacheResponse;
	
	    if (cache != null) {
	      cache.trackResponse(strategy);
	    }
	
	    if (cacheCandidate != null && cacheResponse == null) {
	      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
	    }
	
	    // If we're forbidden from using the network and the cache is insufficient, fail.
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
	
	    // If we don't need the network, we're done.
	    if (networkRequest == null) {
	      return cacheResponse.newBuilder()
	          .cacheResponse(stripBody(cacheResponse))
	          .build();
	    }
	
	    Response networkResponse = null;
	    try {
	      networkResponse = chain.proceed(networkRequest);
	    } finally {
	      // If we're crashing on I/O or otherwise, don't leak the cache body.
	      if (networkResponse == null && cacheCandidate != null) {
	        closeQuietly(cacheCandidate.body());
	      }
	    }
	
	    // If we have a cache response too, then we're doing a conditional get.
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
	
	        // Update the cache after combining headers but before stripping the
	        // Content-Encoding header (as performed by initContentStream()).
	        cache.trackConditionalCacheHit();
	        cache.update(cacheResponse, response);
	        return response;
	      } else {
	        closeQuietly(cacheResponse.body());
	      }
	    }
	
	    Response response = networkResponse.newBuilder()
	        .cacheResponse(stripBody(cacheResponse))
	        .networkResponse(stripBody(networkResponse))
	        .build();
	
	    if (cache != null) {
	      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
	        // Offer this request to the cache.
	        CacheRequest cacheRequest = cache.put(response);
	        return cacheWritingResponse(cacheRequest, response);
	      }
	
	      if (HttpMethod.invalidatesCache(networkRequest.method())) {
	        try {
	          cache.remove(networkRequest);
	        } catch (IOException ignored) {
	          // The cache cannot be written.
	        }
	      }
	    }
	
	    return response;
	  }
	  
核心: 
1. 检查是否有匹配的本地缓存,如果有,可以本地返回,如果没有,则进行下一个节点的处理


逻辑:
1. 从缓存中获取cache
2. 检查cache是否可用
3. 根据检查的结果,如果都不存在,返回错误
4. 没有网络请求时,直接返回缓存
5. 下个chain,处理结果
6. 如果需要,关闭缓存流
7. 有缓存结果,并且网络返回是没有更改的,关闭网络流,记录增加了命中,更新缓存,返回结果
8. 可以写入缓存时,将结果写入到缓存中,返回结果
9. 在不可以写入缓存时,直接返回结果


##### 5.2.3. ConnectInterceptor的处理逻辑


	
	  @Override public Response intercept(Chain chain) throws IOException {
	    RealInterceptorChain realChain = (RealInterceptorChain) chain;
	    Request request = realChain.request();
	    StreamAllocation streamAllocation = realChain.streamAllocation();
	
	    // We need the network to satisfy this request. Possibly for validating a conditional GET.
	    boolean doExtensiveHealthChecks = !request.method().equals("GET");
	    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
	    RealConnection connection = streamAllocation.connection();
	
	    return realChain.proceed(request, streamAllocation, httpCodec, connection);
	  }
	  
	  


##### 5.2.4 网络调用开始前的拦截器,默认没有

##### 5.2.5 CallServerInterceptor 网络处理器


	  @Override public Response intercept(Chain chain) throws IOException {
	    RealInterceptorChain realChain = (RealInterceptorChain) chain;
	    HttpCodec httpCodec = realChain.httpStream();
	    StreamAllocation streamAllocation = realChain.streamAllocation();
	    RealConnection connection = (RealConnection) realChain.connection();
	    Request request = realChain.request();
	
	    long sentRequestMillis = System.currentTimeMillis();
	
	    realChain.eventListener().requestHeadersStart(realChain.call());
	    httpCodec.writeRequestHeaders(request);
	    realChain.eventListener().requestHeadersEnd(realChain.call(), request);
	
	    Response.Builder responseBuilder = null;
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
	
	    httpCodec.finishRequest();
	
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
	
	    realChain.eventListener()
	        .responseHeadersEnd(realChain.call(), response);
	
	    int code = response.code();
	    if (forWebSocket && code == 101) {
	      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
	      response = response.newBuilder()
	          .body(Util.EMPTY_RESPONSE)
	          .build();
	    } else {
	      response = response.newBuilder()
	          .body(httpCodec.openResponseBody(response))
	          .build();
	    }
	
	    if ("close".equalsIgnoreCase(response.request().header("Connection"))
	        || "close".equalsIgnoreCase(response.header("Connection"))) {
	      streamAllocation.noNewStreams();
	    }
	
	    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
	      throw new ProtocolException(
	          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
	    }
	
	    return response;
	  }
	  
	  
