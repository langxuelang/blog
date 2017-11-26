#volley源码学习

>之前一直对于源码学习抱着一种又爱又恨的心情。爱的是因为知道源码有一些特别好的设计思路，可以让自己借鉴，而且对于设计模式来说是最好的实战场。那为啥还会恨呢，曾经很多次下载了很多开源库的源码，可是看的看的就感觉云里雾里，不知所踪。心中没有一个总体的框架，总感觉看的细如牛毛，一叶障目。今天又找时间翻出最简单的volley，准备从头再看一遍。没想到收获很多，写下这篇文章，用来记录。

##volley是什么
volley是一个封装好的网络库，是把httpclient或HttpURLConnection又封装了一层，加上了线程，队列，缓存等机制，让网络请求更加容易。

volley是google官方推出的一个开源项目，专门适用于android轻量的请求。但是对于大文件等支持不好。

volley整个源码都是用了接口编程的思想，这也比较符合设计模式的优秀实践。

所以看volley源码会对于接口编程有较深入的理解。

除了volley之外，还有一个retrofit网络框架，是对okhttp的封装,这个框架现在特别火，而且如果你想领略设计模式之美的话，这个retrofit源码必须看，等之后再写一篇分析retrofit的文章

##volley工作流程

首先先放一张google官方给的流程图，虽然比较简单，但是可以在心中有一个大致的概念

![1.png](https://www.gaotenglife.com/wp-content/uploads/1.png)

首先我们从volley调用入手，找到调用入口，这样就可以按图索骥，一点点摸索出整个的流程。

	RequestQueue queue = Volley.newRequestQueue(this);
	String url ="https://www.gaotenglife.com";
	
	// Request a string response from the provided URL.
	StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
	            new Response.Listener<String>() {
	    @Override
	    public void onResponse(String response) {
	        // Display the first 500 characters of the response string.
	        mTextView.setText("Response is: "+ response.substring(0,500));
	    }
	}, new Response.ErrorListener() {
	    @Override
	    public void onErrorResponse(VolleyError error) {
	        mTextView.setText("That didn't work!");
	    }
	});
	// Add the request to the RequestQueue.
	queue.add(stringRequest);

从上面的代码可以看出，我们首先通过newRequestQueue创建出一个requestqueue，也就是请求队列。然后把一个请求加入到请求队列里面。这里可以看出，当我们把stringrequest加入请求队列后，请求便开始了，所以我们便从add看起。

    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }
        mCacheQueue.add(request);
        return request;
     }

这里面主要的功能就是把网络请求加入到对应的网络请求队列(mNetworkQueue),如果设置了请求需要缓存的话，同时也加入到缓存队列中。这里比较奇怪，为啥只是单单加入队列网络请求就能发出去呢。于是我们接着向下看，最有可能是在mNetworkQueue.add的时候，发出请求。于是我们继续深入,发现mNetworkQueue也只是一个简单的线程安全的队列，也没有做过多的操作。其实这样也是合理的，队列不应该包含更多的业务逻辑在里面。于是我继续查找，是哪里持有了这个队列。

	 public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }

于是发现两个地方使用了这个mNetworkQueue队列，CacheDispatcher，NetworkDispatcher从名字是就能看出，最有可能就是NetworkDispatcher里面进行网络队列的处理了。于是我们看NetworkDispatcher，发现这个NetworkDispatcher原来就是一个线程。

	public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            try {
                processRequest();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
            }
        }
    }

	
在它的run方法里面，不断循环调用processRequest，在processRequest里面从请求队列里取出request
	
	Request<?> request = mQueue.take();//取出请求
	......

	
	 // Perform the network request.
            NetworkResponse networkResponse = mNetwork.performRequest(request);//通过networt真正去请求网络


从这里看出，单一职责设计思想。队列只处理队列的逻辑，网络只处理网络的逻辑。不交叉写到一起。

那么network里面是如何真正请求网络的呢，因为比较volley是封装了具体网络请求库。
于是我们看到BasicNetwork里面，它里面有BaseHttpStack，而BaseHttpStack是个抽象类，它的子类，比如HurlStack是封装了
HttpURLConnection，而HttpClientStack是封装了httpcliet底层库。这样network就可以根据配置，选取我们需要的底层网络库。


那么请求完之后，返回的结果如何回调回去呢。这里又借助ResponseDelivery，将请求的结果发送到ui线程

     mDelivery.postResponse(request, response);

我们再进入ExecutorDelivery这里面

	public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }
然后通过mResponsePoster这里面调用handler，最后将结果发送给ui线程

	 // Deliver a normal response or error, depending.
           if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

这里最后调用了request里面设置的监听函数，最后把数据给了调用方。

到此我们的一个调用过程就分析完毕了。是不是挺简单的。


##缓存的实现

在刚刚上面分析代码的时候说到，mNetworkQueue被两个类持有，一个就是咱们已经分析过的NetworkDispatcher,还有一个就是咱们这一节要讲的CacheDispatcher。

	public <T> Request<T> add(Request<T> request)
		...
		...
		mCacheQueue.add(request);

在上面队列add的时候，如果请求设置了需要缓存，request.shouldCache()，也就是这个为true，那么会先将这个request加入到缓存的队列里面。

而处理缓存队列的，就是CacheDispatcher。和NetworkDispatcher类似，这个CacheDispatcher也是一个线程，在线程的run方法里面，执行了processRequest这个方法。这里面主要处理了缓存相关逻辑

首先先从缓存队列里面将缓存的request取出来。

	final Request<?> request = mCacheQueue.take();
	
	....省略非主要代码
	 // Attempt to retrieve this item from cache.
        Cache.Entry entry = mCache.get(request.getCacheKey());
        if (entry == null) {
            request.addMarker("cache-miss");
            // Cache miss; send off to the network dispatcher.
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                mNetworkQueue.put(request);
            }
            return;
        }

        // If it is completely expired, just send it to the network.
        if (entry.isExpired()) {
            request.addMarker("cache-hit-expired");
            request.setCacheEntry(entry);
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                mNetworkQueue.put(request);
            }
            return;
        }
然后进行了两个判断，第一个就是先从缓存中获取这个request，如果没有缓存，则将请求直接加入到之前的网络队列，进行网络请求。如果有缓存，则再通过判断entry.isExpired是否过期，如果缓存的request已经过期，则也加入到网络请求的队列中。

接下来，如果有缓存并且缓存没有过期，则从缓存中取到之前请求过的数据，并进行解析。如下

	 Response<?> response = request.parseNetworkResponse(
                new NetworkResponse(entry.data, entry.responseHeaders));


之后又通过了一层判断
 
     if (!entry.refreshNeeded()) {
            // Completely unexpired cache hit. Just deliver the response.
            mDelivery.postResponse(request, response);
        } else {
            // Soft-expired cache hit. We can deliver the cached response,
            // but we need to also send the request to the network for
            // refreshing.
            request.addMarker("cache-hit-refresh-needed");
            request.setCacheEntry(entry);
            // Mark the response as intermediate.
            response.intermediate = true;

            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                // Post the intermediate response back to the user and have
                // the delivery then forward the request along to the network.
                mDelivery.postResponse(request, response, new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mNetworkQueue.put(request);
                        } catch (InterruptedException e) {
                            // Restore the interrupted status
                            Thread.currentThread().interrupt();
                        }
                    }
                });
            } else {
                // request has been added to list of waiting requests
                // to receive the network response from the first request once it returns.
                mDelivery.postResponse(request, response);
            }
        }

也就是说，如果缓存的数据需要刷新，那么还是需要将request发送给网络队列进行请求。如果数据不需要刷新，则直接通过mDelivery将缓存的数据发送给ui线程。

这里面有两个概念，如下

	  /** True if the entry is expired. */
        public boolean isExpired() {
            return this.ttl < System.currentTimeMillis();
        }

        /** True if a refresh is needed from the original data source. */
        public boolean refreshNeeded() {
            return this.softTtl < System.currentTimeMillis();
        }

ttl和softttl这两个是http协议里面通过header计算出来的两个值。详细可以查看HttpHeaderParser这个类。


##重试策略

看到重试策略的时候，我首先自己想了下，如果要我自己实现重试策略，我会如何做呢，很直白的的思维就是，在网络请求失败的时候，判断是否有重试策略，然后在网络请求失败的地方，重新发起网络请求。

所以我就一直按照这个思路去寻找，可以在volley网路失败的地方，我只找到了如下的代码

	private static void attemptRetryOnException(String logPrefix, Request<?> request,
            VolleyError exception) throws VolleyError {
        RetryPolicy retryPolicy = request.getRetryPolicy();
        int oldTimeout = request.getTimeoutMs();

        try {
            retryPolicy.retry(exception);
        } catch (VolleyError e) {
            request.addMarker(
                    String.format("%s-timeout-giveup [timeout=%s]", logPrefix, oldTimeout));
            throw e;
        }
        request.addMarker(String.format("%s-retry [timeout=%s]", logPrefix, oldTimeout));
    }

这个函数是所有网络有有异常的时候，会调用的。但是我发现这里面除了设置了重试策略的一些属性，其他没有做网络请求操作。这就奇怪了。难道网络请求会自己发起。这让我百思不得其解。

于是我又一遍一遍看了网络请求的代码，终于发现了端倪

原来在networkdispater调用Network进行网络请求的时候，network里面竟然写了一个while循环

	 @Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
		.....
		.....
		}
	}
这样也就能解释了，这个循环会一直尝试去请求网络，直到不满足重试策略之后，退出循环。也就是说，这个重试策略没有正真参与具体了重试逻辑。而只是保存了自己的重试状态，真正的重试逻辑还是网络请求去保证。
看到这里，我突然感觉到写这样代码的人，真是思路别具一格。而且这样的好处显而易见，你可以重写重试策略，而不需要重新修改网络重试的逻辑。这也就是设计模式里面策略模式比较好的运用吧。


##结论体会

通过完整的看了一遍volley源码，体会到了几个比较重要的思想

* 一个就是单一职责，每一层都分开，负责每一层应该有的功能。互相解耦，底层不依赖上层。比如网络请求Network这个类，不依赖与上层的队列类,而具体封装底层网络请求的BaseHttpStack也不依赖上层Network。可以让network层很轻松的更换底层网络请求库。

* 策略模式的运用，也让代码逻辑与策略分离，策略里面不依赖具体逻辑。逻辑代码里面通过改变不同策略对象，来达到控制不同策略的目的。

* 面向接口的编程，整个volley各个模块都是通过接口互相之间调用，这样不依赖与具体实现，就将整个框架都搭建好了，感觉很受启发。
 






