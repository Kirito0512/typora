### 1. 源码

先放上源码，方便看

```java
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
```



### 2. 整体流程

- 通过chain.request，从cache类中取出cacheCandidate(候选缓存)
- 通过chain.request 和 cacheCandidate,创建 CacheStrategy 缓存策略。并从中拿到networkRequest和cacheResponse
- 如果networkRequest和cacheResponse都为null，返回一个504code的response--------END
- 如果只有networkRequest为null，将缓存返回---------END
- 否则，networkRequest不为null，使用`networkRequest = chain.proceed(networkRequest)` 将请求传递给下一个Interceptor。
- 如果cacheResponse也不为null
  - networkResponse code为304，返回CacheResponse ---------------- END
  - 否则判断是否符合缓存条件，可以的话缓存并将其返回，否则直接返回---------------------END



### 3. 流程细节

 #### 3.1 get cacheCandidate

- 

> 这应该是整个流程中最复杂的部分了，代码只有下面简洁的一句话

```
Response cacheCandidate = cache != null ? cache.get(chain.request()): null;
```

需要先解释下，cache的由来, Cache.java 在 okhttp 的jar包里。

Okhttp默认配置下，cache为null。如果想要用，可以按照下面的例子使用：

```
//缓存文件夹
File cacheFile = new File(getExternalCacheDir().toString(),"cache");
//缓存大小为10M
int cacheSize = 10 * 1024 * 1024;
//创建缓存对象
Cache cache = new Cache(cacheFile,cacheSize);
OkHttpClient client = new OkHttpClient.Builder().cache(cache).build();
```

我们重点来看Cache的get方法源码，行数不多

```
@Nullable Response get(Request request) {
  String key = key(request.url());
  DiskLruCache.Snapshot snapshot;
  Entry entry;
  try {
  	// 注意这里的cache是DiskLruCache.java, 也在Okhttp的jar包里
    snapshot = cache.get(key);
    if (snapshot == null) {
      return null;
    }
  } catch (IOException e) {
    // Give up because the cache cannot be read.
    return null;
  }

  try {
    entry = new Entry(snapshot.getSource(ENTRY_METADATA));
  } catch (IOException e) {
    Util.closeQuietly(snapshot);
    return null;
  }

  Response response = entry.response(snapshot);

  if (!entry.matches(request, response)) {
    Util.closeQuietly(response.body());
    return null;
  }

  return response;
}
```

看一下DiskLruCache的get方法，会返回一个Snapshot对象,分为以下几步

1. initialize()

> 主要是配置journalFile文件 和 lruEntries (LinkedHashMap<String, Entry>).

2. 验证key符合条件
3. 从lruEntries中取出key对应的Entry对象，从Entry中取出Snapshot
4. 向journalFile中添加一行READ数据
5. 返回Snapshot对象



回到源码第8行，接下来

使用Snapshot对象中，长度为2的sources数组，构建Response并返回。

其中ENTRY_METADATA和ENTRY_BODY分别为0和1，代表请求报文的head和body信息。